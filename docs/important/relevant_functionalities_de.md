# Relevante Funktionalitäten
Dieses Kapitel beschreibt die Features und mögliche Implementierungsideen/-Lösungen derer Funktionalitäten, die überhaupt ein echtes Dogu ausmachen und wiederkehrende Probleme lösen. 

Die folgenden Abschnitte beschäftigen sich daher mit wiederkehrenden Funktionen, wie ein Dogu sich in die Landschaft des Cloudogu EcoSystem einbetten kann.

## Authentifizierung

Sofern Dogus dies bereitstellen, können Benutzer:innen die Vorteile von Single Sign-on (SSO) und Single Logout (SLO) auf Enterprise-Niveau genießen. Die Authentifizierung wird hierbei durch den Central Authentication Service (CAS) ermöglicht. Dogu-Entwickler:innen stehen hierbei mehrere Authentifizierungsmöglichkeiten zur Verfügung. Die folgenden drei Abschnitte beschäftigen sich mit Methoden, die bereits erfolgreich eingesetzt wurden, nämlich:

- Authentifizierung mittels CAS-Protokoll 
- Authentifizierung mittels OAuth 
- Authentifizierung mittels OpenID Connect

Grundlage für CAS ist hier immer eine korrekte Konfiguration eines Benutzer-/Gruppenverzeichnisses, sei es das [LDAP](https://github.com/cloudogu/ldap)-Dogu oder ein externer LDAP-kompatibler Dienst.

### CAS-Protokoll

Authentifizierung innerhalb des Cloudogu EcoSystem geschieht über das [CAS-Protokoll](https://apereo.github.io/cas/6.5.x/protocol/CAS-Protocol.html), das Single Sign-on und Single Log-out ermöglicht. Es werden unterschiedliche CAS-Protokoll-Versionen unterstützt (2.0 und 3.0). Bei einer Umsetzung mittels CAS-Protokoll wird empfohlen, die Version 3.0 zu verwenden, da (verglichen mit Version 2.0) die Service-Ticket-Validierung wichtige Benutzerattribute zurückliefern kann.

Folgendes Diagramm zeigt die Beteiligten an der Authentifizierungskonfiguration. Bevor ein Dogu (hier am Beispiel von Redmine) an diesem Prozess teilnehmen kann, muss das Dogu intern einen Satz von CAS-URLs konfigurieren:

- CAS log-in URL (zur Weiterleitung von Benutzenden an das Webformular)
- CAS validation URL (zur Validierung von Service Tickets)
- CAS log-out URL (zur Invalidierung einer SSO-Session)
- Die Service-Erkennung in CAS gegenüber des Dogus geschieht automatisch während der Dogu-Installation und findet im Folgenden keine weitere Betrachtung.

![Beteiligten an der Authentifizierungskonfiguration](../images/important/chapter3_auth_cas_config.png)

Die tatsächliche Authentifizierung geschieht über eine Abfolge von HTTP-Redirects und den Austausch von Session-Cookies im Hintergrund, von denen Benutzende nichts wahrnehmen. Die Anzeige des CAS-Login-Formulars als aktiver Anmeldeschritt sticht dabei stark heraus.

Eine grobe Übersicht über den Anmeldeprozess mit den Beteiligten bietet die folgende Abbildung.

![Workflow der CAS-Authentifizierung](../images/important/chapter3_auth_cas_sequencediag.png)

Das SSO des CAS reduziert diesen Prozess bei der Anmeldung bei weiteren Dogus deutlich.

Weitere Informationen und eine genauere Abbildung vor, während und nach einer Authentifizierung bietet die [CAS-Dokumentation](https://apereo.github.io/cas/6.5.x/protocol/CAS-Protocol.html).

### OAuth-Protokoll

CAS bietet OAuth/OIDC als Protokoll zur Authentifizierung samt SSO/SSL an. Im Folgenden wird die Spezifikation des OAuth 2.0 Protokolls in CAS beschrieben.

#### OAuth Service Account für Dogu erstellen

Damit ein Dogu die OAuth-Endpunkte des CAS benutzen kann, muss sich dieser beim CAS als Client anmelden.
Dafür kann die Aufforderung eines CAS-Service Account in der `dogu.json` des betreffenden Dogus hinterlegt werden.

**Eintrag für einen OAuth Client:**
``` json
"ServiceAccounts": [
    {
        "Type": "cas",
        "Params": [
            "oauth"
        ]
    }
]
```

Die Credentials des Service Accounts werden zufällig generiert (siehe [create-sa.sh](https://github.com/cloudogu/cas/blob/develop/resources/create-sa.sh)) und verschlüsselt in der Registry unter dem Pfad `/config/<dogu>/sa-cas/oauth` und `/config/<dogu>/sa-cas/oauth_client_secret` hinterlegt.

Die Zugangsdaten setzen sich aus der `CLIENT_ID` und dem `CLIENT_SECRET` zusammen. Für den CAS wird das `CLIENT_SECRET` als Hash in der Cloudogu EcoSystem Registry unter dem Pfad `/config/cas/service_accounts/oauth/<CLIENT_ID>` abgelegt.

### OAuth-Endpunkte und Ablauf der Authentifizierung

Die folgenden Schritte beschreiben einen erfolgreichen Ablauf der OAuth-Authentifizierung.

1. Anfordern eines Kurzzeit-Tokens: Siehe Abschnitt unten "OAuth-Authorize-Endpunkt"
2. Kurzzeittoken gegen ein Langzeittoken tauschen: Siehe Abschnitt unten "AccessToken-Endpunkt"
3. Langzeittoken kann nun zu Authentifizierung gegen Ressourcen benutzen werden.
   Derzeit bietet CAS nur das Profil der User als Resource an: Siehe Abschnitt unten "OAuth-Userprofil"

![Ablauf der Authentifizierung mit OAuth 2.0](../images/important/chapter3_auth_oauth_sequencediag.png)

#### OAuth-Authorize-Endpunkt

Dieser Endpunkt dient als initialer Start der OAuth-Authorisation.
Der Authorisation-Endpunkt wird benutzt, um ein kurzlebiges Token vom CAS anzufordern.

**URL** : `<fqdn>/oauth2.0/authorize`

**Method** : `GET`

**Bedingung der Daten**

```
?response_type = code
?client_id     = Valide ClientID von dem Dogu
?state         = Irgendeine Zeichenkette
?redirect_url  = <URL zu die der Kurzzeittoken erfolgreicher Authentifizierung weitergeleitet wird>
```

**Daten-Beispiel**

```
?response_type = code
?client_id     = portainer
?state         = b8c57125-9281-4b67-b857-1559cdfcdf31
?redirect_url  = http://local.cloudogu.com/portainer/
```

**Aufruf Beispiel**

```
https://local.cloudogu.com/cas/oauth2.0/authorize?client_id=portainer&redirect_uri=http%3A%2F%2Flocal.cloudogu.com%2Fportainer%2F&response_type=code&state=b8c57125-9281-4b67-b857-1559cdfcdf31
```

##### Erfolgreiche Antwort

Leitet einen automatisch zur CAS-Login-Maske.
Nach erfolgreichem Login wird die `redirect_url` mit einem `code` als GET-Parameter übergeben.

Beispiel für `code`: `ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local`

#### OAuth-Access-Token

Dieser Endpunkt dient zum Austausch eines Kurzzeittokens (`code`) gegen ein Langzeittoken (`access_token`).

**URL** : `<fqdn>/oauth2.0/accessToken`

**Method** : `GET`

**Data constraints**

```
?grant_type    = athorization_code
?code          = Valider Code vom `authorize` Endpunkt
?client_id     = Valide ClientID von dem Dogu
?client_secret = Valides Secret von dem Dogu
?redirect_url  = <URL zu die der Langzeittoken erfolgreicher Authentifizierung geschickt wird>
```

**Data example**

```
?grant_type    = athorization_code
?code          = ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local
?client_id     = portainer
?client_secret = sPJtcNrmROZ3sZu3
?redirect_url  = https://local.cloudogu.com/portainer/
```

**Call example**

```
https://local.cloudogu.com/cas/oauth2.0/accessToken?grant_type=authorization_code&code=ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local&client_id=portainer&client_secret=sPJtcNrmROZ3sZu3&redirect_uri=https%3A%2F%2Flocal.cloudogu.com%2Fportainer%2F
```

##### Erfolgreiche Antwort

**Status:** 200 OK

**Beispiel-Antwort:**

``` json
{
    "access_token": "TGT-1-m2gUNJwEqXyV7aAEXekihcVnFc5iI4mpfdZGOTSiiHzEbwr1cr-cas.ces.local",
    "expires_in": "7196",
    "token_type": "Bearer"
}
``` 

##### Nicht-Erfolgreiche Antwort

**Fehler:** Der Kurzzeittoken ist invalid oder schon abgelaufen.

**Status:** 500 OK

**Beispiel-Antwort:**

``` json
{
    "message": "invalid_grant"
}
```

#### OAuth-Userprofil

Dieser Endpunkt dient zur Abfrage des Userprofils vom eingeloggten User mithilfe eines Langzeittokens (`access_token`).

**URL** : `<fqdn>/oauth2.0/profile`

**Method** : `GET`

**Request-Header**

```
authorization = Valider Access Token als Bearer
```

**Request-Header Beispiel**

```
authorization: Bearer TGT-1-m2gUNJwEqXyV7aAEXekihcVnFc5iI4mpfdZGOTSiiHzEbwr1cr-cas.ces.local
```

##### Erfolgreiche Antwort

**Status:** 200 OK

**Beispiel-Antwort:**

``` json
{
  "id": "cesadmin",
  "attributes": {
    "username": "cesadmin",
    "cn": "admin",
    "mail": "cesadmin@localhost.de",
    "givenName": "admin",
    "surname": "admin",
    "displayName": "admin",
    "groups": [
      "cesManager",
      "cesadmin"
    ]
  }
}
``` 

##### Nicht-Erfolgreiche Antwort

**Fehler:** Der Langzeittoken ist invalid oder schon abgelaufen.

**Status:** 500 OK

**Beispiel-Antwort:**

``` json
{
    "message": "expired_accessToken"
}
```

### OpenID Connect-Protokoll

CAS bietet OAuth/OpenID Connect (OIDC) als Protokoll zur Authentifizierung samt SSO/SSL an. Im Folgenden wird die Spezifikation des OpenID Connect-Protokolls in CAS beschrieben. 

**Vorsicht!**

Dieser Abschnitt ist noch unzureichend geklärt, insbesondere hinsichtlich der Ähnlichkeit zur Authentifizierung durch OAuth 2.0. Pull-Requests durch die Community sind hier gern gesehen. :blue_heart:

#### OIDC Service Account für Dogu erstellen

Damit ein Dogu die OIDC-Endpunkte des CAS benutzen kann, muss sich dieser beim CAS als Client anmelden.
Dafür kann die Aufforderung eines CAS-Service Account in der `dogu.json` des betreffenden Dogus hinterlegt werden.

**Eintrag für einen OIDC Client:**
``` json
"ServiceAccounts": [
    {
        "Type": "cas",
        "Params": [
            "oidc"
        ]
    }
]
```

Die Credentials des Service Accounts werden zufällig generiert (siehe [create-sa.sh](https://github.com/cloudogu/cas/blob/develop/resources/create-sa.sh)) und verschlüsselt in der Registry unter dem Pfad `/config/<dogu>/sa-cas/oidc` und `/config/<dogu>/sa-cas/oidc_client_secret` hinterlegt.

Die Zugangsdaten setzen sich aus der `CLIENT_ID` und dem `CLIENT_SECRET` zusammen. Für den CAS wird das `CLIENT_SECRET` als Hash in der Cloudogu EcoSystem Registry unter dem Pfad `/config/cas/service_accounts/oidc/<CLIENT_ID>` abgelegt.

## Auf die Registry zugreifen

Die Cloudogu EcoSystem Registry ist eine Key-Value-Datenbank, die auch für Dogus ein Kernelement darstellt. Die Konfiguration eines Dogus wird über die Registry durchgeführt. Darüber hinaus werden in der Registry werden Werte abgelegt, die auch von globaler Natur sind.

### Registry-Zugriff vom Dogu heraus

Das folgende Bild fokussiert Teile, die in der Kommunikation zwischen Dogu (exemplarisch hier: Redmine) und der Registry eine Rolle spielen:

![Beteiligte Komponenten, die im Zusammenspiel zwischen Dogu und Registry eine Rolle spielen](../images/important/chapter3_ces_registry_components.png)

- Registry: Sie läuft jenseits des Dogu-Containers und ist über eine IP-Adresse aus dem Container-Netzwerk erreichbar
  - Nach der Installation enthält die Registry neben anderen Keys den Public Key jedes Dogus.
- Datei `/etc/ces/node_master` 
  - Diese wird während der Dogu-Installation automatisch in das Dogu gemountet. Sie enthält die IP-Adresse der Registry, damit das Dogu auf die Registry zugreifen kann.
- Datei `/private/private.pem`
  - Diese Datei enthält den Private Key des Dogus. Dieser wird u. a. zum Entschlüsseln von verschlüsselten Werten der Registry verwendet.
  - Häufig handelt es sich hierbei um [Service-Accounts](../core/compendium_de.md#serviceaccounts) zu anderen Dogus

Die Dogu-spezifische Konfiguration liegt im Registry-Pfad `/config/<dogu>/`.

Eine wertvolle Hilfe ist das Kommandozeilenwerkzeug `doguctl` unter anderem auch in der Startphase des Dogu-Containers. Dieses Werkzeug vereinfacht den Zugriff auf die Registry, indem automatisch die `node_master`-Datei ausgelesen wird oder wie Dogu-eigene Registry-Schlüssel adressiert werden.

Die `dogu.json` erlaubt es, eigene Konfigurationswerte zu definieren, die sogar validiert werden können.  

```bash
# liest Konfigurationwert aus dem Schlüssel /config/<dogu>/my_key
doguctl config my_key
my old value

# liest globalen Konfigurationwert aus /config/_global/fqdn
doguctl config -g fqdn
your-ecosystem.example.com

# liest Konfigurationwert aus /config/<dogu>/my_key, gibt einen Defaultwert zurück, wenn dieser nicht gesetzt wurde
doguctl config --default NOVALUE my_key2
NOVALUE

# liest verschlüsselten DB-Namen /config/<dogu>/sa-postgresql/db_name
doguctl config -e sa-postgresql/db_name
your-dogu-database-1234

# schreibt Konfigurationswert in /config/<dogu>/my_key hinein
doguctl config my_key 'my new value'

# schreibt verschlüsselten Geheimnis /config/<dogu>/geheim/credential
doguctl config -e geheim/credential '$up3r$3cre7'
```

### Interessante Registryzweige

Es existieren jenseits der Dogu-eigenen Registrywerte noch weitere Bereiche, die im Dogu-Betrieb von Interesse sind.

Die globale Konfiguration liegt im Registry-Pfad `/config/_global/` und kann mit `doguctl` (wie oben gezeigt) verwaltet werden.

- Globale Werte `/config/_global`
  - `/config/_global/fqdn`
    - die FQDN dieser Cloudogu EcoSystem-Instanz
    - z. B. eco.example.com
    - Dieser Wert kann sich im laufenden Betrieb ändern. Siehe hierzu auch den Abschnitt zur [Änderbarkeit der FQDN](#änderbarkeit-der-fqdn)
  - `/config/_global/domain`
    - der Mail-Domänen-Anteil dieser Cloudogu EcoSystem-Instanz
    - z. B. example.com
  - `/config/_global/mail_address`
    - die Emailadresse des Instanzadministrators
  - `/config/_global/admin_group`
    - der aktuelle Name der LDAP-Gruppe, deren Mitglieder die Clodogu EcoSystem-Instanz in der UI administrieren
    - Dieser Wert kann sich im laufenden Betrieb ändern. Siehe hierzu auch den Abschnitt zur [Änderbarkeit der Admingruppe](#änderbarkeit-der-admin-gruppe)
- Doguzustände `/state/<dogu>`
  - wenn das Dogu einen [HealthCheck](../core/compendium_de.md#healthchecks) vom Typen `state` definiert, dann ermöglicht es dem Administrator und anderen Dogus Health-Hinweise auf das Dogu zu erhalten
  - Im eigenen Dogu z. B. `doguctl state installing` setzen, wenn eine längere Installationsroutine gestartet wird. Kurz bevor der Hauptprozess gestartet wird dann mit `doguctl state ready` einen ordnungsgemäßen Betriebszustand anzeigen.
  - Im EcoSystem-Host lässt sich dies mit `cesapp healthy <dogu>` überprüfen
  - In anderen Dogus lässt sich dies mit `doguctl healthy <dogu>` überprüfen
    - Mit dem Schalter `--timeout <Sekunden>` kann man so auf Dogus warten, von denen das eigene Dogu abhängt.
    - Ein Exit-Code ungleich null (0) deutet darauf hin, dass entweder das Dogu nicht rechtzeitig seinen Startvorgang beendet hat oder die Registry-Kommunikation fehlschlug.

## Aufbau und Best Practices von `startup.sh` 

Ein einfaches Dogu benötigt eigentlich nur eine `dogu.json` und ein Container-Image. Doch was geschieht, wenn sich die Wirklichkeit im Dauerbetrieb von den Annahmen während der Dogu-Entwicklung unterscheiden? 

Bei Cloudogu haben wir schnell festgestellt, dass es im Betrieb geschickter ist, auf solche Änderungen eingehen zu können. Das Spektrum ist hierbei breit. Dabei kann es sich um Wartezeiten gegenüber anderen Dogus handeln, oder um die Änderung von ursprünglich fixen Bezeichnern bis hin zu Störungen in der Container-Engine. 

Wiederkehrende Aufgaben beim Container-Start befinden sich im Abschnitt "Typische Dogu-Features" in den Abschnitten:
- [Änderbarkeit der Admingruppe](#änderbarkeit-der-admin-gruppe)
- [Änderbarkeit der FQDN](#änderbarkeit-der-fqdn)
- [Loglevel mappen und ändern](#loglevel-mappen-und-ändern)

Um dynamisch auf diese Gegebenheiten zu reagieren hat es sich bei Cloudogu eingebürgert, nicht den Hauptprozess im Container direkt zu starten. Stattdessen werden erst Fehlerfälle in einem Startskript abgeprüft, ggf. neue Konfigurationswerte einarbeitet, um dann erst den Hauptprozess zu starten, z. B. so:

1. Container startet `startup.sh`
2. `startup.sh`
   1. setzt eigene Optionen
   2. ruft eigene Kommandos auf, um bestimmte Funktionalitäten umzusetzen, um z. B.
      - auf Dogu-Abhängigkeiten zu warten
      - einmalige Installationsprozesse durchzuführen 
      - ein temporäres Admin-Konto zu generieren
      - das aktuelle Log-Level umsetzen
   3. den Dogu-State auf `ready` setzen (geeigneter [HealthCheck](../core/compendium_de.md#healthchecks) vorausgesetzt)
   4. startet den Hauptprozess

Dieser Abschnitt geht daher auf Erkenntnisse und _Best Practices_ ein, die sich auf solche Startskripte beziehen: Die `startup.sh`. 

Übrigens: Fleißige Entwickler:innen können Inspiration in den Cloudogu-eigenen Startskripten [z. B. im Redmine-Dogu](https://github.com/cloudogu/redmine/blob/develop/resources/startup.sh) sammeln.

### Skriptinterpreter

Um ein Skript in einem Dogu ausführen zu können, muss ein Skriptinterpreter im Container-Image existieren. Das kann ein offizielles Paket sein (wie `bash`), es spricht aber nichts dagegen seinen eigenen Skriptinterpreter zu verwenden. Da Bash-Skripte weit verbreitet sind, wird hier Bash-Syntax verwendet.

### Fehlerbehandlung

Je nach Dogu können Fehler an unterschiedlichen Stellen entstehen und (dummerweise) auch wieder verschluckt werden. Es ist eine gute Praxis, Fehler hart aufschlagen zu lassen, um die Fehlerursache schneller zu identifizieren.

Hierzu werden als allererstes die folgenden Optionen gesetzt

```bash
#!/bin/bash
set -o errexit # beende das gesamte Skript (und damit auch den Container) bei einem nicht abgefangenem Fehler 
set -o nounset # finde nicht initialisierte Variablen
set -o pipefail # verschlucke keine Fehler bei Pipe-Verwendung
```

Wie ein Fehler abgefangen wird, ist ein Implementierungsdetail. Manchmal ist es günstig, einen Fehlerfall zu testen und eine eigene Meldung abzusetzen, manchmal reicht der Fehler selbst aus.

Wenn man bereits Fehler erwartet, dann kann dieses Konstrukt hilfreich sein:

```bash
yourCommandExitCode=0
your-command || yourCommandExitCode=$?
if [[ ${yourCommandExitCode} -ne 0 ]]; then
  echo "ERROR: Oh no. I found something critical during QuirkyFunction.";
  doguctl state "ErrorQuirkyFunction"
  sleep 300
  exit 1
fi
```

Der Fehler des Kommandos `your-command` wird abgefangen. Ein eigener Fehlertext wird ausgegeben. Der Dogu-State wird auf einen Nicht-`ready`-Wert gesetzt, der beim Debugging hilfreich sein könnte. Schließlich wird fünf Minuten (= 300 Sekunden) gewartet bis mit `exit 1` das Dogu als fehlerhaft neu gestartet wird.

### Bash `function` - Teile und herrsche

Je länger man eine Anwendung entwickelt, desto mehr Funktionalitäten gelangen in die Anwendung. Genauso kann es sich auch mit der `startup.sh` verhalten. Von einer leichteren Testbarkeit (z. B. mit [BATS](https://github.com/bats-core/bats-core)) abgesehen, sind kleine Ausführungsbestandteile leichter verständlich.

Es ist daher eine gute Idee, Dogu-Funktionalitäten in der `startup.sh` genauso unter Gesichtspunkten der Testbarkeit, Lesbarkeit oder Refaktorisierbarkeit zu gestalten.

Hierbei helfen Bash-Funktionen:

```bash
function setDoguLogLevel() {
  echo "Mapping dogu specific log level..."
  currentLogLevel=$(doguctl config --default "WARN" "logging/root")
  
  # bilde hier das Loglevel auf die Logkonfiguration des Dogus ab 
}
```

Wiederkehrende Funktionen können abstrahiert und parametrisiert werden:

```bash
function deleteSetupArtefact() {
  local artefactFile="${1}"
  rm -f "${artefactFile}"
}

function cleanUpSetup() {
  deleteSetupArtefact "/tmp/tracker-extract-3-files.127"
  deleteSetupArtefact "/usr/share/lib/intermediateFile.bin"
}
```

Mit steigender Komplexität ist es evtl. eine Idee wert, relevante Schritte mit einem `echo` zu versehen, um im Fehlerfall eine Suche nach dem Fehler zu beschleunigen.

### Die Nutzung von `doguctl`

Der Abschnitt [über Registry-Zugriff](#registry-zugriff-vom-dogu-heraus) hat das Thema `doguctl` bereits angeschnitten. `doguctl` ist ein Kommandozeilenwerkzeug, das wiederkehrende Interaktionen mit dem Cloudogu EcoSystem bündelt und vereinfacht. Dieser Abschnitt beschreibt mögliche Aufrufe. 

#### doguctl config

Liest und schreibt Konfigurationswerte aus der Registry

#### doguctl validate

Validiert Konfigurationswerte der Registry

#### doguctl healthy

Prüft, ob ein gegebenes Dogu betriebsbereit (healthy) ist

#### doguctl state

Liest und schreibt Dogu-Zustandswerte.

#### doguctl random

Erzeugt Zufallsstrings, geeignet um Passwörter oder sonstige Zugangsdaten zu erzeugen.

#### doguctl template

Erzeugt eine Datei aus einem [Golang-Template](https://pkg.go.dev/text/template). Registrywerte und Umgebungsvariablen können hierin direkt verwendet werden.

#### doguctl wait-for-tcp

Wartet bis ein gegebener TCP-Port offen ist.

#### doguctl wait-for-http

Wartet bis eine gegebene HTTP-URL bereit ist.


## Service Accounts
Lorem ipsum Einführungstext

### Service Accounts produzieren 


### Service Accounts konsumieren

## Dogu-Upgrades


## Typische Dogu-Features

Dieses Kapitel beschreibt Funktionen, die Dogus tiefer in das Cloudogu EcoSystem integrieren und sie einheitlich administrierbar machen.

### Memory-/Swap-Limit

Mit Memory- und Swap-Limits kann man den Speicherverbrauch (Arbeitsspeicher und Swap) von Dogus einschränken.

Falls ein Dogu seine Speicherlimitierung überschreitet, wird der größte Prozess im Container beendet.
Dies ist normalerweise der Hauptprozess des Dogus und führt dazu, dass der Container neu gestartet wird.

Wird kein Wert bei der Memory-Limitierung gesetzt, findet diese auch nicht statt.
Bei der Swap-Limitierung is `0b` der Standardwert und stellt somit keinen Swap zur Verfügung.

Um eine Limitierung vornehmen zu können, muss die dogu.json des Dogus folgende Einträge enthalten:

```json
{
    "Name": "container_config/memory_limit",
    "Description":"Limits the container's memory usage. Use a positive integer value followed by one of these units [b,k,m,g] (byte, kibibyte, mebibyte, gibibyte).",
    "Optional": true,
    "Validation": {
    "Type": "BINARY_MEASUREMENT"
    }
},
{
    "Name": "container_config/swap_limit",
    "Description":"Limits the container's swap memory usage. Use zero or a positive integer value followed by one of these units [b,k,m,g] (byte, kibibyte, mebibyte, gibibyte). 0 will disable swapping.",
    "Optional": true,
    "Validation": {
    "Type": "BINARY_MEASUREMENT"
    }
}
```

Hiermit lassen sich die etcd-Einträge `container_config/memory_limit` und `container_config/swap_limit` in der jeweiligen Dogu-Konfiguration erstellen.

Die konfigurierbaren Werte für die Schlüssel sind jeweils eine Zeichenkette der Form `<Zahlenwert><Einheit>` und beschreiben die maximal vom Dogu nutzbare Menge an Speicher.
Zu beachten ist hier, dass zwischen dem numerischen Wert und der Einheit kein Leerzeichen stehen darf.
Verfügbare Einheiten sind `b`, `k`, `m` und `g` (für byte, kibibyte, mebibyte und gibibyte).

Das Setzen der Werte kann über folgende Wege erfolgen:
- `doguctl config container_config/memory_limit 1g`
- `cesapp edit-config <doguname>` (nur vom Host aus)
- `etcdctl set /config/<doguname>/container_config/memory_limit "1g"` (nur vom Host aus)

Um die Limitierungen zu übernehmen, muss das Dogu neu erstellt (`cesapp recreate <doguname>`) und anschließend neu gestartet (`cesapp start <doguname>`) werden.

Ein Sonderfall stellt die Limitierung eines Java-Prozesses dar. Enthält ein Dogu einen Java-Prozess, können folgende zusätzliche Einträge in die dogu.json eingebaut werden:

```json
{
  "Name": "container_config/java_max_ram_percentage",
  "Description": "Limits the heap stack size of the Java process to the configured percentage of the available physical memory when the container has more than approx. 250 MB of memory available. Is only considered when a memory_limit is set. Use a valid float value with decimals between 0 and 100 (f. ex. 55.0 for 55%). Default value: 25%",
  "Optional": true,
  "Default": "25.0",
  "Validation": {
    "Type": "FLOAT_PERCENTAGE_HUNDRED"
  }
},
{
  "Name": "container_config/java_min_ram_percentage",
  "Description": "Limits the heap stack size of the Java process to the configured percentage of the available physical memory when the container has less than approx. 250 MB of memory available. Is only considered when a memory_limit is set. Use a valid float value with decimals between 0 and 100 (f. ex. 55.0 for 55%). Default value: 50%",
  "Optional": true,
  "Default": "50.0",
  "Validation": {
    "Type": "FLOAT_PERCENTAGE_HUNDRED"
  }
}
```

Die damit konfigurierbaren Werte müssen in den Start-Skripten des Dogus dem entsprechenden Java-Prozess als Parameter mitgegeben werden. Eine Referenzimplementierung findet sich im [Nexus-Dogu](https://github.com/cloudogu/nexus/blob/77bdcfdbe0787c85d2d9b168dc38ff04b225706d/resources/util.sh#L52).



### Backup & Restore-Fähigkeit

Ein Feature des Cloudogu EcoSystems ist, dass alle Dogus so ausgelegt sind, dass sie sich über ein zentrales Backupsystem sichern und wiederherstellen lassen. Damit das funktioniert, müssen alle Dogus ihre veränderlichen Daten in Volumes auslagern. Dieses Vorgehen wird bereits im [Compendium](../core/compendium_de.md#volumes) beschrieben.

Bei der Entwicklung eines Dogus ist darauf zu achten, dass das Dogu nach einem erfolgreichen Backup- und Restore-Vorgang weiterhin normal funktioniert und alle Daten und Funktionen vorhanden sind. Dazu müssen zuerst alle Volumes in der dogu.json, die Produktivdaten enthalten, mit dem [NeedsBackup](../core/compendium_de.md#needsbackup)-Flag gekennzeichnet werden. Danach sollte das Dogu gebaut und mit Testdaten gefüllt werden.
Anschließend führt man ein Backup des Systems und danach einen Restore durch. Dazu können die Befehle [cesapp backup](https://docs.cloudogu.com/de/docs/system-components/cesapp/operations/backup/) und [cesapp restore](https://docs.cloudogu.com/de/docs/system-components/cesapp/operations/restore/) genutzt werden.

Sind alle Dogus wieder hochgefahren, testet man, ob das eigene Dogu normal läuft und alle Testdaten weiterhin vorhanden sind.


### Änderbarkeit der Admin-Gruppe

Bei der Installation des CES wird eine globale Admin-Gruppe eingestellt. Alle Mitglieder dieser Gruppe sollen in allen Dogus Admin-Rechte bekommen. Dies muss bei der Entwicklung des Dogus sichergestellt werden. Der Name der global definierten Admin-Gruppe lässt sich aus dem Dogu heraus mit `doguctl` abfragen: `doguctl config --global 'admin_group'`

Zusätzlich kann die Admin-Gruppe im Nachhinein geändert werden. Das Dogu muss dann (ggf. nach einem Neustart) so reagieren, dass nur noch die Mitglieder der neuen Admin-Gruppe Admin-Rechte erhalten. Nutzer in der alten Admin-Gruppe erhalten nur noch normale Nutzerrechte, sofern sie sich nicht auch in der neuen Admin-Gruppe befinden.


### Änderbarkeit der FQDN


### Loglevel mappen und ändern

