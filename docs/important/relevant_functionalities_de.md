# Relevante Funktionalitäten
Dieses Kapitel beschreibt die Features und mögliche Implementierungsideen/-Lösungen derer Funktionalitäten, die überhaupt ein echtes Dogu ausmachen und wiederkehrende Probleme lösen.

Die folgenden Abschnitte beschäftigen sich daher mit wiederkehrenden Funktionen, wie ein Dogu sich in die Landschaft des Cloudogu EcoSystem einbetten kann.

## Authentifizierung

Sofern Dogus dies bereitstellen, können Benutzer:innen die Vorteile von Single Sign-on (SSO) und Single Logout (SLO) auf Enterprise-Niveau genießen. Die Authentifizierung wird hierbei durch den Central Authentication Service (CAS) ermöglicht. Dogu-Entwickler:innen stehen hierbei mehrere Authentifizierungsmöglichkeiten zur Verfügung. Die folgenden drei Abschnitte beschäftigen sich mit Methoden, die bereits erfolgreich eingesetzt wurden, nämlich:

- Authentifizierung mittels CAS-Protokoll 
- Authentifizierung mittels OAuth 2.0-Protokoll 
- Authentifizierung mittels OpenID Connect-Protokoll

Grundlage für CAS ist hier immer eine korrekte Konfiguration eines Benutzer-/Gruppenverzeichnisses, sei es das [LDAP](https://github.com/cloudogu/ldap)-Dogu oder ein externer LDAP-kompatibler Dienst.

### CAS-Protokoll

Authentifizierung innerhalb des Cloudogu EcoSystem geschieht über das [CAS-Protokoll](https://apereo.github.io/cas/6.5.x/protocol/CAS-Protocol.html), das Single Sign-on und Single Log-out ermöglicht. Es werden unterschiedliche CAS-Protokoll-Versionen unterstützt (2.0 und 3.0). Bei einer Umsetzung mittels CAS-Protokoll wird empfohlen, die Version 3.0 zu verwenden, da (verglichen mit Version 2.0) nach einer erfolgreichen Authentifizierung wichtige Benutzerattribute zurückliefern kann.

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

CAS bietet OAuth/OIDC als Protokoll zur Authentifizierung samt SSO/SLO an. Im Folgenden wird die Spezifikation des OAuth 2.0 Protokolls in CAS beschrieben.

#### OAuth Service Account für Dogu erstellen

Damit ein Dogu die OAuth-Endpunkte des CAS benutzen kann, muss sich dieser beim CAS als Client anmelden.
Dafür kann die Anforderung eines OAuth-spezifischen CAS-Service Accounts in der `dogu.json` des betreffenden Dogus hinterlegt werden.

**Eintrag für einen OAuth Client:**
```json
{
  "ServiceAccounts": [
    {
      "Type": "cas",
      "Params": [
        "oauth"
      ]
    }
  ]
}
```

Die Credentials des Service Accounts werden zufällig generiert (siehe [create-sa.sh](https://github.com/cloudogu/cas/blob/develop/resources/create-sa.sh)) und verschlüsselt in der Registry unter dem Pfad `/config/<dogu>/sa-cas/oauth` und `/config/<dogu>/sa-cas/oauth_client_secret` hinterlegt.

Die Zugangsdaten setzen sich aus der `CLIENT_ID` und dem `CLIENT_SECRET` zusammen. Für den CAS wird das `CLIENT_SECRET` als Hash in der Cloudogu EcoSystem Registry unter dem Pfad `/config/cas/service_accounts/oauth/<CLIENT_ID>` abgelegt.

### OAuth-Endpunkte und Ablauf der Authentifizierung

Die folgenden Schritte beschreiben einen erfolgreichen Ablauf der OAuth-Authentifizierung.

1. Anfordern eines Kurzzeit-Tokens: Siehe Abschnitt unten "OAuth-Authorize-Endpunkt"
2. Kurzzeittoken gegen ein Langzeittoken tauschen: Siehe Abschnitt unten "AccessToken-Endpunkt"
3. Langzeittoken kann nun zur Authentifizierung gegen Ressourcen benutzen werden.
   Derzeit bietet CAS nur das Profil der User als Resource an: Siehe Abschnitt [OAuth-Userprofil](#oauth-userprofil)

![Ablauf der Authentifizierung mit OAuth 2.0](../images/important/chapter3_auth_oauth_sequencediag.png)

#### OAuth-Authorize-Endpunkt

Dieser Endpunkt dient als initialer Start der OAuth-Autorisation.
Der `authorize`-Endpunkt wird benutzt, um ein kurzlebiges Token vom CAS anzufordern.

**URL** : `<fqdn>/oauth2.0/authorize`

**Method** : `GET`

**Bedingung der Daten**

```
?response_type = code
?client_id     = Valide ClientID von dem Dogu
?state         = Irgendeine Zeichenkette
?redirect_url  = <URL, zu die der Kurzzeittoken bei erfolgreicher Authentifizierung weitergeleitet wird>
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

Leitet den User automatisch zur CAS-Login-Maske weiter.
Nach erfolgreichem Login wird die `redirect_url` als `code` GET-Parameter übergeben.

Beispiel für `code`: `ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local`

#### OAuth-Access-Token

Dieser Endpunkt dient zum Austausch eines Kurzzeittokens (`code`) gegen einen Langzeittoken (`access_token`).

**URL** : `<fqdn>/oauth2.0/accessToken`

**Method** : `GET`

**Data constraints**

```
?grant_type    = authorization_code
?code          = Valider Code vom `authorize` Endpunkt
?client_id     = Valide ClientID von dem Dogu
?client_secret = Valides Secret von dem Dogu
?redirect_url  = <URL, zu die der Langzeittoken bei erfolgreicher Authentifizierung geschickt wird>
```

**Data example**

```
?grant_type    = authorization_code
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

```json
{
    "access_token": "TGT-1-m2gUNJwEqXyV7aAEXekihcVnFc5iI4mpfdZGOTSiiHzEbwr1cr-cas.ces.local",
    "expires_in": "7196",
    "token_type": "Bearer"
}
```

##### Nicht-Erfolgreiche Antwort

**Fehler:** Der Kurzzeittoken ist ungültig oder schon abgelaufen.

**Status:** 500 Bad request

**Beispiel-Antwort:**

```json
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

```json
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

**Fehler:** Der Langzeittoken ist ungültig oder schon abgelaufen.

**Status:** 500 Bad request

**Beispiel-Antwort:**

```json
{
    "message": "expired_accessToken"
}
```

#### OAuth-Logout-Endpunkt

Dieser Endpunkt dient zur Beendigung der OAuth-Session.
Der Logout-Endpunkt wird benutzt, um den Langzeit-Token vom CAS zu invalidieren.

**URL** : `<fqdn>/cas/logout`

### OpenID Connect-Protokoll

CAS bietet OAuth/OpenID Connect (OIDC) als Protokoll zur Authentifizierung samt SSO/SLO an. Im Folgenden wird die Spezifikation des OpenID Connect-Protokolls in CAS beschrieben.

#### OIDC Service Account für Dogu erstellen

Damit ein Dogu die OIDC-Endpunkte des CAS benutzen kann, muss sich dieser beim CAS als Client anmelden.
Dafür kann die Anforderung eines OIDC-spezifischen CAS-Service Accounts in der `dogu.json` des betreffenden Dogus hinterlegt werden.

**Eintrag für einen OIDC Client:**
```json
{
  "ServiceAccounts": [
    {
      "Type": "cas",
      "Params": [
        "oidc"
      ]
    }
  ]
}
```

Die Credentials des Service Accounts werden zufällig generiert (siehe [create-sa.sh](https://github.com/cloudogu/cas/blob/develop/resources/create-sa.sh)) und verschlüsselt in der Registry unter dem Pfad `/config/<dogu>/sa-cas/oidc` und `/config/<dogu>/sa-cas/oidc_client_secret` hinterlegt.

Die Zugangsdaten setzen sich aus der `CLIENT_ID` und dem `CLIENT_SECRET` zusammen. Für den CAS wird das `CLIENT_SECRET` als Hash in der Cloudogu EcoSystem Registry unter dem Pfad `/config/cas/service_accounts/oidc/<CLIENT_ID>` abgelegt.

#### OIDC-Authorize-Endpunkt

Dieser Endpunkt dient als initialer Start der OpenID Connect-Autorisation.
Der `authorize`-Endpunkt wird benutzt, um ein kurzlebiges Token vom CAS anzufordern.

**URL** : `<fqdn>/oidc/authorize`

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
?client_id     = teamscale
?state         = b8c57125-9281-4b67-b857-1559cdfcdf31
?redirect_url  = http://local.cloudogu.com/teamscale/
```

**Aufruf Beispiel**

```
https://local.cloudogu.com/cas/oidc/authorize?client_id=portainer&redirect_uri=http%3A%2F%2Flocal.cloudogu.com%2Fteamscale%2F&response_type=code&state=b8c57125-9281-4b67-b857-1559cdfcdf31
```

##### Erfolgreiche Antwort

Leitet einen automatisch zur CAS-Login-Maske.
Nach erfolgreichem Login wird die `redirect_url` mit einem `code` als GET-Parameter übergeben.

Beispiel für `code`: `ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local`

#### OIDC-Access-Token

Dieser Endpunkt dient zum Austausch eines Kurzzeittokens (`code`) gegen ein Langzeittoken (`access_token`).

**URL** : `<fqdn>/oidc/accessToken`

**Method** : `GET`

**Data constraints**

```
?grant_type    = authorization_code
?code          = Valider Code vom `authorize` Endpunkt
?client_id     = Valide ClientID von dem Dogu
?client_secret = Valides Secret von dem Dogu
?redirect_url  = <URL zu die der Langzeittoken erfolgreicher Authentifizierung geschickt wird>
```

**Data example**

```
?grant_type    = authorization_code
?code          = ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local
?client_id     = teamscale
?client_secret = sPJtcNrmROZ3sZu3
?redirect_url  = https://local.cloudogu.com/teamscale/
```

**Call example**

```
https://local.cloudogu.com/cas/oidc/accessToken?grant_type=authorization_code&code=ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local&client_id=teamscale&client_secret=sPJtcNrmROZ3sZu3&redirect_uri=https%3A%2F%2Flocal.cloudogu.com%2Fteamscale%2F
```

##### Erfolgreiche Antwort

**Status:** 200 OK

**Beispiel-Antwort:**

```json
{
    "access_token": "TGT-1-m2gUNJwEqXyV7aAEXekihcVnFc5iI4mpfdZGOTSiiHzEbwr1cr-cas.ces.local",
    "expires_in": "7196",
    "token_type": "Bearer"
}
```

##### Nicht-Erfolgreiche Antwort

**Fehler:** Der Kurzzeittoken ist invalid oder schon abgelaufen.

**Status:** 500 Bad request

**Beispiel-Antwort:**

```json
{
    "message": "invalid_grant"
}
```

#### OIDC-Logout-Endpunkt

Dieser Endpunkt dient zur Beendigung der OpenID Connect-Session.
Der Logout-Endpunkt wird benutzt, um den Langzeit-Token vom CAS zu invalidieren.

**URL** : `<fqdn>/oidc/logout`

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
?client_id     = teamscale
?state         = b8c57125-9281-4b67-b857-1559cdfcdf31
?redirect_url  = http://local.cloudogu.com/teamscale/
```

**Aufruf Beispiel**

```
https://local.cloudogu.com/cas/oidc/authorize?client_id=teamscale&redirect_uri=http%3A%2F%2Flocal.cloudogu.com%2Fteamscale%2F&response_type=code&state=b8c57125-9281-4b67-b857-1559cdfcdf31
```

##### Erfolgreiche Antwort

Leitet einen automatisch zur CAS-Login-Maske.
Nach erfolgreichem Login wird die `redirect_url` mit einem `code` als GET-Parameter übergeben.

Beispiel für `code`: `ST-1-wzG237MUOvfjfZrvRH5s-cas.ces.local`

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
  - Häufig handelt es sich hierbei um [Service-Accounts](../core/compendium_de.md#serviceaccounts) zu anderen Dogus.

Die Dogu-spezifische Konfiguration liegt im Registry-Pfad `/config/<dogu>/`. [Registry-Schlüssel](../core/compendium_de.md#configuration) sollten im `snake_case` geschrieben werden, also Lowercase mit Unterstrichen.

Eine wertvolle Hilfe ist das Kommandozeilenwerkzeug `doguctl` unter anderem auch in der Startphase des Dogu-Containers. Dieses Werkzeug vereinfacht den Zugriff auf die Registry, z.B. durch das automatische Auslesen der `node_master`-Datei oder die einfache Adressierung von Dogu-eigenen Registry-Schlüsseln. Eine Auflistung der genauen Kommandos ist unter [Die Nutzung von doguctl](#die-nutzung-von-doguctl) zu finden.

Die `dogu.json` erlaubt es, eigene Konfigurationswerte zu definieren, die auch automatisch validiert werden können.

```bash
# liest Konfigurationwert aus dem Schlüssel /config/<dogu>/my_key
$ doguctl config my_key
my old value

# liest globalen Konfigurationwert aus /config/_global/fqdn
$ doguctl config -g fqdn
your-ecosystem.example.com
```

### Weitere Registryeinträge

Es existieren jenseits der doguspezifischen Registrywerte noch weitere Bereiche, die in der Dogu-Startphase von Interesse sind.

Die globale Konfiguration liegt im Registry-Pfad `/config/_global/` und kann mit [doguctl](#die-nutzung-von-doguctl) verwaltet werden.

- Globale Werte `/config/_global`
  - `/config/_global/fqdn`
    - die FQDN dieser Cloudogu EcoSystem-Instanz
    - z. B. eco.example.com
    - Dieser Wert kann sich im laufenden Betrieb ändern. Siehe hierzu auch den Abschnitt zur [Änderbarkeit der FQDN](#änderbarkeit-der-fqdn)
  - `/config/_global/domain`
    - der Domänen-Anteil dieser Cloudogu EcoSystem-Instanz, den Dogus verwenden können, um Mails zu versenden.
    - z. B. example.com
  - `/config/_global/mail_address`
    - die Emailadresse des Instanzadministrators
    - z. B. petra.mustermann@example.com
  - `/config/_global/admin_group`
    - der aktuelle Name der LDAP-Gruppe, deren Mitglieder die Cloudogu EcoSystem-Instanz in der UI administrieren
    - Dieser Wert kann sich im laufenden Betrieb ändern. Siehe hierzu auch den Abschnitt zur [Änderbarkeit der Admingruppe](#änderbarkeit-der-admin-gruppe)
- Doguzustände `/state/<dogu>`
  - wenn das Dogu einen [HealthCheck](../core/compendium_de.md#healthchecks) vom Typen `state` definiert, dann ermöglicht es dem Administrator und anderen Dogus Health-Hinweise auf das Dogu zu erhalten
  - Im eigenen Dogu z. B. `doguctl state installing` setzen, wenn eine längere Installationsroutine gestartet wird. Kurz bevor der Hauptprozess gestartet wird dann mit `doguctl state ready` einen ordnungsgemäßen Betriebszustand anzeigen.
  - Im EcoSystem-Host lässt sich dies mit `cesapp healthy <dogu>` überprüfen
  - In anderen Dogus lässt sich dies mit `doguctl healthy <dogu>` überprüfen
    - Mit dem Schalter `--timeout <Sekunden>` kann man so auf Dogus warten, von denen das eigene Dogu abhängt.
    - Ein Exit-Code ungleich Null (0) deutet darauf hin, dass entweder das Dogu nicht rechtzeitig seinen Startvorgang beendet hat oder die Registry-Kommunikation fehlschlug.

## Aufbau und Best Practices von `startup.sh`

Ein einfaches Dogu benötigt eigentlich nur eine `dogu.json` und ein Container-Image. Doch was geschieht, wenn sich die Wirklichkeit im Dauerbetrieb von den Annahmen während der Dogu-Entwicklung unterscheiden?

Bei Cloudogu haben wir schnell festgestellt, dass es im Betrieb geschickter ist, auf solche Änderungen eingehen zu können. Das Spektrum ist hierbei breit. Dabei kann es sich um Wartezeiten gegenüber anderen Dogus handeln, oder um die Änderung von ursprünglich fixen Bezeichnern bis hin zu Störungen in der Container-Engine.

Wiederkehrende Aufgaben beim Container-Start befinden sich im Abschnitt "Typische Dogu-Features" in den Abschnitten:
- [Änderbarkeit der Admingruppe](#änderbarkeit-der-admin-gruppe)
- [Änderbarkeit der FQDN](#änderbarkeit-der-fqdn)
- [Logging-Verhalten steuern](#logging-verhalten-steuern)

Um dynamisch auf diese Gegebenheiten zu reagieren, hat es sich bei Cloudogu eingebürgert, nicht den Hauptprozess im Container direkt zu starten. Stattdessen werden erst Fehlerfälle in einem Startskript abgeprüft, ggf. neue Konfigurationswerte gesetzt, um dann erst den Hauptprozess zu starten, z. B. so:

1. Container startet `startup.sh`
2. `startup.sh`
   1. setzt eigene Optionen
   2. ruft eigene Kommandos auf, um bestimmte Funktionalitäten umzusetzen, um z. B.
      - auf Dogu-Abhängigkeiten zu warten
      - einmalige Installationsprozesse durchzuführen 
      - ein temporäres Admin-Konto zu generieren
      - das aktuelle Log-Level umzusetzen
      - Registrieren der Cloudogu EcoSystem-TLS-Zertifikate in der Software/Zielsystem
      - Systemänderungen der Registry mit API-Zugriffen verarbeiten, um die Software vorzubereiten
        - Replikation von LDAP-Gruppen / Rechten
      - Einstellen der Software (Log-Level, sonstige Konfiguration, z. B. durch `doguctl template`)
      - Auf Systemänderungen der Registry reagieren: z. B. [Admin-Gruppe](#änderbarkeit-der-admin-gruppe) hat sich geändert, was im Dogu weitere Maßnahmen benötigt
   3. den Dogu-State auf `ready` setzen (geeigneter [HealthCheck](../core/compendium_de.md#healthchecks) vorausgesetzt)
   4. startet den Hauptprozess

Dieser Abschnitt geht daher auf Erkenntnisse und _Best Practices_ ein, die sich auf solche Startskripte beziehen: Die `startup.sh`.

Übrigens: Fleißige Entwickler:innen können Inspiration in den Cloudogu-eigenen Startskripten [z. B. im Redmine-Dogu](https://github.com/cloudogu/redmine/blob/develop/resources/startup.sh) sammeln.

### Skriptinterpreter

Um ein Skript in einem Dogu ausführen zu können, muss ein Skriptinterpreter im Container-Image existieren. Das kann ein offizielles Paket sein (wie `bash`), es spricht aber nichts dagegen seinen eigenen Skriptinterpreter zu verwenden. Da Bash-Skripte weit verbreitet sind, wird hier Bash-Syntax verwendet.

### Fehlerbehandlung

Je nach Dogu können Fehler an unterschiedlichen Stellen entstehen und (dummerweise) auch wieder verschluckt werden. Es ist eine gute Praxis, das Skript bei Fehlern mit einem entsprechenden Exit-Code abzubrechen, um die Fehlerursache schneller zu identifizieren.

Hierzu werden als allererstes die folgenden Optionen gesetzt

```bash
#!/bin/bash
set -o errexit # beende das gesamte Skript (und damit auch den Container) bei einem nicht abgefangenen Fehler 
set -o nounset # finde nicht initialisierte Variablen
set -o pipefail # verschlucke keine Fehler bei Pipe-Verwendung
```

Wie ein Fehler abgefangen wird, ist ein Implementierungsdetail. Manchmal ist es günstig, einen Fehlerfall zu testen und eine eigene Meldung abzusetzen, manchmal reicht der Fehler selbst aus.

Wenn man bereits Fehler erwartet, dann kann dieses Konstrukt hilfreich sein:

```bash
yourCommandExitCode=0
your-command || yourCommandExitCode=$?
if [[ ${yourCommandExitCode} -ne 0 ]]; then
  echo "ERROR: Oh no. An error occurred during the execution of your-command.";
  doguctl state "ErrorYourFunctionality"
  sleep 300
  exit 1
fi
```

Der Fehler des Kommandos `your-command` wird abgefangen. Ein eigener Fehlertext wird ausgegeben. Der Dogu-State wird auf einen Nicht-`ready`-Wert gesetzt, der beim Debugging hilfreich sein könnte. Schließlich wird fünf Minuten (= 300 Sekunden) gewartet bis mit `exit 1` das Dogu als fehlerhaft neu gestartet wird.

### Teile und herrsche mit Bash

Je länger man eine Anwendung entwickelt, desto mehr Funktionalitäten gelangen in die Anwendung. Genauso kann es sich auch mit der `startup.sh` verhalten. Von einer leichteren Testbarkeit (z. B. mit [BATS](https://github.com/bats-core/bats-core)) abgesehen, sind kleine Ausführungsbestandteile leichter verständlich.

Es ist daher eine gute Idee, Dogu-Funktionalitäten in der `startup.sh` genauso unter Gesichtspunkten der Testbarkeit, Lesbarkeit oder Refaktorisierbarkeit zu gestalten.

#### Shell functions

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
function deleteSetupArtifact() {
  local artifactFile="${1}"
  rm -f "${artifactFile}"
}

function cleanUpSetup() {
  deleteSetupArtifact "/tmp/tracker-extract-3-files.127"
  deleteSetupArtifact "/usr/share/lib/intermediateFile.bin"
}
```

#### Nachvollziehbarkeit während der Startphase

Mit steigender Komplexität ist es eine Idee wert, relevante Schritte mit einem `echo` zu versehen, um im Fehlerfall eine Suche nach dem Fehler in den Logausgaben zu beschleunigen.

Zeitgleich gibt es mit dem [Dogu-State-HealthCheck](../core/compendium_de.md#healthchecks) die Möglichkeit, während des Dogu-Starts eigene Ausführungsphasen zu markieren. Zum Beispiel `doguctl state 'aSpecialInstallPhase'` am Anfang oder `doguctl state 'ready'` am Ende des Skripts.

Ein HealthCheck vom Typ `state` wird ausschließlich dann erfolgreich sein, wenn der state den String `ready` enthält.

#### Auslagerung von Code

Eine weitere Möglichkeit ist die Auslagerung von Funktionen in andere Skriptdateien, die dann mittels `source` in das eigentliche Startskript inkludiert werden. Es sollte allerdings beachtet werden, dass auch ein solches `source`n fehlschlagen kann und behandelt werden sollte.

```bash
sourcingExitCode=0
source /resources/util.sh || sourcingExitCode=$?
if [[ ${sourcingExitCode} -ne 0 ]]; then
  echo "ERROR: An error occurred while sourcing /resources/util.sh: Exit code ${sourcingExitCode}"
  doguctl state "ErrorSourcingUtilSh"
  sleep 300
  exit 1
fi
```

### Qualitätssicherung von Startskripten

Bei komplexeren Aufgaben kann es sein, dass ein Startskript wie die `startup.sh` an Komplexität gewinnt. Wie jedes andere Programm sollte es getestet und auf Robustheit analysiert werden. Hierbei gibt es mehrere Möglichkeiten. [BATS](https://github.com/bats-core/bats-core) kann im Unit-Test von Shell-Funktionen unterstützen.

Analysewerkzeuge wie [Shellcheck](https://www.shellcheck.net/) können Fehler in der Benutzung von Shell-Aufrufen aufdecken.

### Die Nutzung von `doguctl`

Der Abschnitt [über Registry-Zugriff](#registry-zugriff-vom-dogu-heraus) hat das Thema `doguctl` bereits angeschnitten. `doguctl` ist ein Kommandozeilenwerkzeug, das wiederkehrende Interaktionen mit der Cloudogu EcoSystem-Registry bündelt und vereinfacht. Dieser Abschnitt beschreibt mögliche Aufrufe.

Mit `--help` gibt jedes Unterkommando von `doguctl` eine Hilfeseite aus.

#### doguctl config

Dieser Aufruf liest und schreibt Konfigurationswerte.

```bash
# liest Konfigurationwert aus dem Schlüssel /config/<dogu>/my_key
$ doguctl config my_key
my old value

# liest globalen Konfigurationwert aus /config/_global/fqdn
$ doguctl config -g fqdn
your-ecosystem.example.com

# liest Konfigurationwert aus /config/<dogu>/my_key und gibt einen Defaultwert zurück, wenn dieser nicht gesetzt wurde
$ doguctl config --default NOVALUE my_key2
NOVALUE

# liest verschlüsselten DB-Namen /config/<dogu>/sa-postgresql/db_name
$ doguctl config -e sa-postgresql/db_name
your-dogu-database-1234

# schreibt Konfigurationswert in /config/<dogu>/my_key hinein
$ doguctl config my_key 'my new value'

# schreibt verschlüsseltes Geheimnis /config/<dogu>/geheim/credential
$ doguctl config -e geheim/credential '$up3r$3cre7'

# löscht den Schlüssel /config/<dogu>/delete_me
$ doguctl config --rm delete_me
```

#### doguctl validate

Dieser Aufruf validiert Konfigurationswerte der Registry. Das Kommando `validate` liefert einen Exit-Code == 0, wenn alle HealthChecks des betrachteten Dogus ein positives Ergebnis zurückgeben. Ansonsten liefert es einen Exit-Code == 1.

```bash
$ doguctl validate logging/root # validiert einzelnen Wert, Gutfall: Wert=WARN aus ERROR,WARN,INFO,DEBUG
Key logging/root: OK
$ echo $?
0

$ doguctl validate logging/root #  validiert einzelnen Wert, Fehlerfall: Wert=foo aus ERROR,WARN,INFO,DEBUG
Key logging/root: INVALID
at least one key was invalid
$ echo $?
1

$ doguctl validate --all # validiert alle Werte mit Ausgabe
...
Key container_config/memory_limit: OK
Key container_config/swap_limit: OK
Key container_config/java_max_ram_percentage: OK
Key container_config/java_min_ram_percentage: OK
Key logging/root: INVALID
Key caching/workdir/size: OK (No Validator)
at least one key was invalid
$ echo $?
1

$ doguctl validate --silent # validiert Werte ohne Ausgabe (kombinierbar mit --all), Gutfall
$ echo $?
0

$ doguctl validate --silent # validiert Werte ohne Ausgabe (kombinierbar mit --all), Fehlerfall
at least one key was invalid
echo $?
1
```

#### doguctl healthy

Dieser Aufruf prüft, ob ein gegebenes Dogu betriebsbereit (healthy) ist. Das Kommando `healthy` liefert einen Exit-Code == 0, wenn das betrachtete Dogu healthy ist. Ansonsten liefert es einen Exit-Code == 1.

```bash
if ! doguctl healthy --wait --timeout 120 postgresql; then
  echo "timeout reached by waiting of ldap to get healthy"
  exit 1
fi
```

#### doguctl state

Dieser Aufruft liest und schreibt Dogu-Zustandswerte. Er wird in Kombination mit einem [HealthCheck](../core/compendium_de.md#healthchecks) verwendet.

```bash
$ doguctl state "installing" # schreibt den Wert in den State
# Erledige Installation

$ doguctl state # Liest den Wert aus dem State
installing

$ doguctl state "ready" # Setzt den State auf den Standard-Healthy-Wert
# Starte nun die Anwendung
```

#### doguctl random

Dieser Aufruf erzeugt Zufallsstrings, geeignet um Passwörter oder sonstige Zugangsdaten zu erzeugen.

```bash
$ doguctl random # erzeugt Zufallsstring mit der Länge 16
9HoF4nYmlLYtf6Ju
$ doguctl random  -l 5 # erzeugt Zufallsstring mit Länge 5 
W6Wmj
```

#### doguctl template

Dieser Aufruf erzeugt eine Datei aus einem [Golang-Template](https://pkg.go.dev/text/template). Unterschiedliche Registrywerte und Umgebungsvariablen können hierin direkt verarbeitet werden.

Der Aufruf nimmt ein bis zwei Parameter entgegen:
```bash
$ doguctl template <Template-Datei> [Ausgabedatei]
```

Wird das Kommando `template` ohne Ausgabedatei verwendet, dann wird die gerenderte Ausgabe auf `stdout` ausgegeben.

Unterstützte Template-Parameter:

- Umgebungsvariablen
  - `.Env.Get <Umgebungsvariable>` - verwendet vorher exportierte Umgebungsvariable "ADMIN_USERNAME"
- Dogu-Konfiguration
  - `.Config.Get <Dogu-Konfigurationsschlüssel>` - verwendet unverschlüsselten Dogu-Konfigurationswert unter `/config/<dogu>/<schlüssel>` 
  - `.Config.GetOrDefault <Dogu-Konfigurationsschlüssel> <Defaultwert>` - verwendet unverschlüsselten Dogu-Konfigurationswert oder den bereitgestellten Defaultwert
  - `.Config.GetAndDecrypt <verschlüsselter Dogu-Konfigurationsschlüssel>` - entschlüsselt den Dogu-Konfigurationswert und verwendet ihn
  - `.Config.Exists <Dogu-Konfigurationsschlüssel>` - gibt einen `bool`-Wert zurück, ob ein Dogu-Konfigurationswert unter `/config/<dogu>/<schlüssel>` existiert
- Globale Konfiguration
  - `.GlobalConfig.Get <globaler Konfigurationsschlüssel>` - verwendet globalen Konfigurationswert `/config/_global/<schlüssel>`
  - `.GlobalConfig.GetOrDefault <globaler Konfigurationsschlüssel> <Defaultwert>` - verwendet unverschlüsselten globalen Konfigurationswert oder den bereitgestellten Defaultwert
  - `.GlobalConfig.Exists <Dogu-Konfigurationsschlüssel>` - gibt einen `bool`-Wert zurück, ob ein globaler Konfigurationswert unter `/config/_global/<schlüssel>` existiert
- Dogu-Abfragen
  - `.Dogus.IsEnabled <Dogu-Name>` - gibt einen `bool`-Wert zurück, ob ein Dogu installiert ist
  - `.Dogus.Get <Dogu-Name>` - gibt ein [Dogu](../core/compendium_de.md#type-dogu)-Struct zurück
  - `.Dogus.GetAll` - gibt ein Slice aller [Dogus](../core/compendium_de.md#type-dogu) zurück

Beispiel einer Template-Datei:

```gotemplate
# my-pseudo-config.conf
[Global]
admin.username = {{ .Env.Get "ADMIN_USERNAME" }}
funny.name = {{ .Config.Get "something_funny" }}
log.level = {{ .Config.GetOrDefault "log_level" "WARN" }}
url.jdbc = jdbc:postgresql://postgresql:5432/{{ .Config.GetAndDecrypt "sa-postgresql/database" }}
url.fqdn = https://{{ .GlobalConfig.Get "fqdn" }}/my-dogu

{{ if .Config.Exists "notification" }}
notification = mail
{{ end }}

{{ if .Dogus.IsEnabled "redmine" }}
ticketsystem.url = https://{{ .GlobalConfig.Get "fqdn" }}/redmine
{{ end }}
...
```

#### doguctl wait-for-tcp

Mit diesem Aufruf wartet `doguctl` bis ein gegebener TCP-Port offen ist oder ein Timeout (in Sekunden) eintritt. Ein Exit-Code != 0 signalisiert einen Fehler.

```bash
if ! doguctl wait-for-tcp --timeout 120 8080; then
  echo "Received timeout after waiting 120 seconds for port 8080. Exiting."
  exit 1
fi
```

#### doguctl wait-for-http

Mit diesem Aufruf wartet `doguctl` bis eine gegebene HTTP(s)-URL bereit ist oder ein Timeout (in Sekunden) eintritt. Ein Exit-Code != 0 signalisiert einen Fehler.

```bash
if ! doguctl wait-for-tcp --host postgresql --timeout 120 5432; then
  echo "Received timeout after waiting 120 seconds for port 5432 of host postgresql. Exiting."
  exit 1
fi
```

## Service Accounts

Service Accounts bilden einen Mechanismus für Zugangskonten ab, die Dogus zur Absicherung oder Speicherung ihrer Daten benötigen, diese Funktionalität aber nicht selbst bereitstellen möchten. Dogus können sich als Produzent und/oder als Konsument von Service Accounts darstellen. Bei beiden handelt es sich um ein jeweils optionales Feature eines Dogus.

In der Regel diktiert die Natur der Applikation im Dogu selbst, ob bzw. welches von beiden am sinnvollsten ist. So wäre ein Datenbankmanagement-Dogu wie PostgreSQL nicht hilfreich, das anderen Dogus keine Zugänge zu seiner Datenbasis anbietet. Andererseits wäre es für ein Dogu wie Redmine fatal, kein Dogu zu haben, in das es per SQL seine Daten ablegen kann.

Dogus wie PostgreSQL, die anderen Dogus gegenüber Zugangskonten bereitstellen können, nennt man auch _Service-Account-Produzenten_. Dogus wie Redmine, die ein Zugangskonto bei einem anderen Dogu benötigen, nennt man _Service-Account-Konsumenten_.

### Service Accounts produzieren

Service Accounts werden nicht von einem Service-Account-Konsumenten selbst angefragt, weil ein Dogu sich nicht um diese Form der Orchestrierung kümmern sollte. Dies ist stattdessen die Aufgabe von einem Client wie `cesapp` oder `k8s-dogu-operator` während der Installation des Service-Account-Konsumenten. Der `ExposedCommand` hierfür lautet `service-account-create`.

Ähnlich hierzu sieht die Löschung von Service Accounts aus, denn ebenfalls kümmert sich ein Client wie `cesapp` oder `k8s-dogu-operator` während der Deinstallation eines Dogus darum, dass dessen angeforderte Service Accounts wieder gelöscht werden. Der `ExposedCommand` hierfür lautet `service-account-remove`.

Damit ein Dogu als Service-Account-Produzent auftreten kann, muss es in seiner `dogu.json` zwei ausführbare Dateien bereitstellen:

```json
{
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
  ]
}
```

Die Begriffe `service-account-create` und `service-account-remove` sind geschützte Eigenschaften und werden von Clients, die die `dogu.json` interpretieren, als Befehle zum Erstellen und Löschen von Service Accounts benutzt.

In den beiden folgenden Abschnitten wird auf die Erzeugung und Löschung von Service Accounts eingegangen. Weitere Information über ExposedCommands befinden sich im [ExposedCommands](../core/compendium_de.md#exposedcommands)-Abschnitt des Kompendiums.

#### Service Account anlegen

Das `service-account-create`-Skript verwaltet das Erstellen eines neuen Service Accounts. Es wird immer ausgeführt, wenn ein neuer Konsument installiert wird. Während des Aufrufs erhält das `service-account-create`-Skript jedenfalls den Namen des Konsumenten-Dogus als erstes Argument. Ein zweites Argument zur Feineinstellung des Service Accounts ist optional und hängt von der Unterstützung des Produzenten-Dogus ab.

Ein `service-account-create`-Skript besteht aus drei Schritten:

1. Erstellen der Zugangsdaten
2. Anlegen eines Accounts für unsere Software mit den Informationen
3. Zurückgeben der Zugangsdaten
   - Dies führt dazu, dass sie automatisch und verschlüsselt in den Dogu-spezifischen Konfigurationsbereich des Verbrauchers geschrieben werden.

Ein Beispiel für das `service-account-create`-Skript könnte folgendermaßen aussehen:

`create-sa.sh`

```bash
#!/bin/bash
set -o errexit
set -o nounset
set -o pipefail
doguName="${1}"
{
  #1) Erstellen der Zugangsdaten
  USER="${doguName}-$(doguctl random)"
  PASSWORD=$(doguctl random)

  #2) Legt einen Account für unsere Software mit den Informationen an
  # ... Implementiere die Dogu-spezifische Logik hier
} >/dev/null 2>&1 # unterdrücke sonstige Ausgaben auf stdout, da nur Zugangsdaten oder Fehler erwartet werden

  #3) Gib Zugangsdaten zurück. Der Client kümmert sich um die Speicherung und Verschlüsselung.
echo "username: ${USER}"
echo "password: ${PASSWORD}"
```

In Schritt 1 werden Zugangsdaten zufällig generiert. Der Name des Konsumenten-Dogus kann in die Zugangsdaten eingearbeitet werden, allerdings ist dies keine Pflicht.

In Schritt 2 muss ein Account mit den generierten Informationen in der Dogu-Software erstellt werden.

In Schritt 3 werden die Zugangsdaten mit dem Format `echo "<registrykey>: <registryvalue>"` ausgegeben. `registrykey` und `registryvalue` müssen durch einen Doppelpunkt und einem Leerzeichen getrennt sein. Die gesamte Zeile muss durch einen Zeilenumbruch beendet werden.

Diese Zugangsdaten werden vom verarbeitenden Client automatisch eingelesen und in den Dogu-spezifischen Konfigurationsbereich des Verbrauchers verschlüsselt abgelegt.
Nach dem obigen Beispiel würde das Konsumenten-Dogu je einen Registry-Eintrag für Username und Passwort erzeugen:

- `/config/<konsument>/sa-<produzent>/username`
- `/config/<konsument>/sa-<produzent>/password`

Das Verbaucher-Dogu kann diese nun z. B. durch `doguctl config -e sa-<produzentdogu>/username` auslesen und entschlüsseln.

#### Service Account löschen

Das `service-account-remove`-Skript löscht einen bestehenden Service Account eines Konsumenten-Dogus.

Es wird immer ausgeführt, wenn das Konsumenten-Dogu deinstalliert wird. Während des Aufrufs erhält das `service-account-remove`-Skript jedenfalls den Namen des Konsumenten-Dogus als erstes Argument. Da hier keine Ausgaben des Skriptes verarbeitet werden, können auch Fehlermeldungen auf `stdout` ausgegeben werden.

Ein `service-account-remove`-Skript besteht aus zwei Schritten:

1. Den Konsumenten identifizieren 
2. Löschen der Daten und des Service Accounts des Konsumenten

Ein Beispiel für das `service-account-remove`-Skript könnte folgendermaßen aussehen:

`remove-sa.sh`

```bash
#!/bin/bash
set -o errexit
set -o nounset
set -o pipefail

#1) den Konsumenten identifizieren
doguName="${1}"
if [ X"${doguName}" = X"" ]; then
  echo "usage remove-sa.sh dogu-name"
  exit 1
fi

#2) Löschen des Service Accounts des Verbrauchers
# ... Implementiere die Dogu-spezifische Logik hier
```

In Schritt 1 wird der Name des Konsumenten-Dogus ausgelesen (z. B. "redmine"), um den Service Account des Dogus zu identifizieren.

In Schritt 2 wird der Account aus der Datenbasis des Produzenten inkl. Zugangsdaten gelöscht.

### Service Accounts konsumieren

Es ist sehr einfach, bei einem Produzenten-Dogu einen Service Account anzufragen, da die Hauptarbeit beim Client und beim Produzenten liegt. Ein Konsumenten-Dogu muss lediglich in seiner `dogu.json` das gewünschte Produzenten-Dogu als  [Service-Account](../core/compendium_de.md#serviceaccounts) nennen:

```json
{
  "ServiceAccounts": [
    {
      "Type": "produzentendogu",
      "Kind": "dogu"
    }
  ]
}
```

Bei einem Service Account handelt es sich um eine besondere Form der Abhängigkeit. Daher ist es sinnvoll, das Produzenten-Dogu in seiner [Dependency-Liste](../core/compendium_de.md#dependencies) zu führen. So wird in den unterschiedlichen Phasen einer Dogu-Installation sichergestellt, dass das Produzenten-Dogu wirklich zur Benutzung bereitsteht:

```json
{
  "Dependencies": [
    {
      "Name": "produzentendogu",
      "Type": "dogu"
    }
  ],
  "ServiceAccounts": [
    {
      "Type": "produzentendogu",
      "Kind": "dogu"
    }
  ]
}
```

## Dogu-Upgrades

Dogus können im Cloudogu EcoSystem mit einfachen Mitteln (bspw. `cesapp upgrade dogu <doguname>`) aktualisiert werden. Der gesamte Upgrade-Prozess muss dazu vorher bei der Dogu-Entwicklung automatisiert werden und kann dann von den CES-Tools (hier: `cesapp`) ohne große Zuarbeit der Admins durchlaufen.

Der Upgrade-Prozess enthält u. A. folgende Schritte:
- Anzeigen von Upgrade Notifications, falls vorhanden
- Download des neuen Dogu-Images
- Sicherstellen, dass das Dogu healthy ist
- Sicherstellen, dass alle abhängigen Dogus verfügbar sind
- Ausführen des Pre-Upgrade-Skripts im alten Dogu, falls vorhanden
- Ersetzen des alten Dogus durch das neue
- Starten des neuen Dogus
- Ausführen des Post-Upgrade-Skripts im neuen Dogu, falls vorhanden

Falls es beim Dogu-Upgrade notwendig ist, vor- oder nachbereitende Schritte durchzuführen, können optionale Pre- und Post-Upgrade-Skripte erstellt werden.
Als Beispiel kann man eine Software betrachten, die nach einem Update ein anderes Datenbankschema benötigt.
In einem solchen Fall ist es möglich, Upgrade-Skripte zu schreiben, um eine Migration der Datenbank nach einem Dogu-Upgrade zu starten.

Außerdem kann man über wichtige Änderungen beim Upgrade eines Dogus durch ein Upgrade-Notification-Skript hinweisen lassen.

Alle im Folgenden beschriebenen Upgrade-Skripte sollten (neben bspw. dem `startup.sh`-Skript) im `resources`-Ordner abgelegt werden.

### Ermittlung der Upgrade-Pfade

Bei der Entwicklung der Dogus ist dafür zu sorgen, dass Upgrades eines Dogus auch über mehrere Versionen hinweg funktionieren. Dafür müssen die Upgrade-Skripte so gestaltet sein, dass sie den Upgradepfad ermitteln und ggf. nötige Migrationsschritte automatisch ausführen.

Hierfür können in allen Upgrade-Skripten die vorherige (`FROM`) und neue (`TO`) Dogu-Version ausgelesen werden:

```bash
FROM_VERSION="${1}"
TO_VERSION="${2}"
```

### pre-upgrade - Führt alle Aktionen vor dem Upgrade des Dogus durch

Das Pre-Upgrade-Skript kann als [Exposed Command](../core/compendium_de.md#exposedcommands) in der `dogu.json` eines Dogus definiert werden:

```json
{
  "ExposedCommands": [
    {
      "Name": "pre-upgrade",
      "Command": "/pre-upgrade.sh"
    }
  ]
}
```

Dieses Skript wird vor dem eigentlichen Upgrade des Dogus im alten Dogu-Container ausgeführt.

### post-upgrade - Führt alle Aktionen nach dem Upgrade des Dogus durch

Das Post-Upgrade-Skript kann als [Exposed Command](../core/compendium_de.md#exposedcommands) in der `dogu.json` eines Dogus definiert werden:

```json
{
  "ExposedCommands": [
    {
      "Name": "post-upgrade",
      "Command": "/post-upgrade.sh"
    }
  ]
}
```

Dieses Skript wird nach dem Upgrade des Dogus im neuen Dogu-Container gestartet, sobald dieser hochfährt. Es laufen also im Normalfall sowohl das `startup.sh`- als auch das `post-upgrade`-Skript gleichzeitig los. Falls es notwendig sein sollte, dass das `post-upgrade`-Skript noch vor dem `startup.sh`-Skript ausgeführt wird, muss das `startup.sh`-Skript mit einem Wartemechanismus versehen werden. Dies kann bspw. durch das Warten auf einen Registry-Key geschehen, den das `post-upgrade`-Skript setzt, sobald es komplett durchgelaufen ist.

### upgrade-notification - Zeigt eine Benachrichtigung vor der Upgradebestätigung eines Dogus

Das Upgrade-Notification-Skript kann als [Exposed Command](../core/compendium_de.md#exposedcommands) in der `dogu.json` eines Dogus definiert werden:

```json
{
  "ExposedCommands": [
    {
      "Name": "upgrade-notification",
      "Command": "/upgrade-notification.sh"
    }
  ]
}
```

Dieses Skript wird vor dem Upgrade-Vorgang ausgeführt und sollte ausschließlich Informationen (per `echo`) ausgeben, die für den Administrator vor dem Upgrade relevant sein können. Beispielsweise kann hier auf Breaking Changes hingewiesen werden oder darauf, dass man vor dem Upgrade ein Backup anlegen sollte.

## Typische Dogu-Features

Dieses Kapitel beschreibt Funktionen, die Dogus tiefer in das Cloudogu EcoSystem integrieren und sie einheitlich administrierbar machen.

### Memory-/Swap-Limit

Mit Memory- und Swap-Limits kann man den Speicherverbrauch (Arbeitsspeicher und Auslagerungsspeicher) von Dogus einschränken.

Falls ein Dogu seine Speicherlimitierung überschreitet, wird der größte Prozess im Container beendet.
Dies ist normalerweise der Hauptprozess des Dogus und führt dazu, dass der Container neu gestartet wird.

Wird kein Wert bei der Memory-Limitierung gesetzt, findet diese auch nicht statt.
Bei der Swap-Limitierung ist `0b` der Standardwert und stellt somit keinen Swap zur Verfügung.

#### Vorbereitung der Speicherlimits im Host

Damit der Cloudogu EcoSystem-Host die Limitierung des Speichers **und** Swaps vornehmen kann, müssen vorher folgende Einstellungen vorgenommen werden:
1. die Datei `/etc/default/grub` öffnen
2. Zur Variable `GRUB_CMDLINE_LINUX_DEFAULT` muss folgender Wert hinzugefügt werden:
   `cgroup_enable=memory swapaccount=1`
3. die Änderungen speichern 
4. den Befehl `sudo update-grub` bzw. `sudo update-bootloader --refresh` ausführen 
5. die Maschine neu starten

**Achtung!**
Die obige Aktivierung mit `cgroup_enable=memory swapaccount=1` führt voraussichtlich zu einem Speicher-Overhead von 1 % und einer Performanz-Einbuße von 10 %, selbst wenn Docker nicht läuft.

#### Limitierung im Dogu

Um eine Limitierung vornehmen zu können, muss die `dogu.json` des Dogus folgende Einträge enthalten:

```json
{
  "Configuration": [
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
  ]
}
```

Hiermit lassen sich die Registry-Einträge `container_config/memory_limit` und `container_config/swap_limit` in der jeweiligen Dogu-Konfiguration erstellen.

Die konfigurierbaren Werte für die Schlüssel sind jeweils eine Zeichenkette der Form `<Zahlenwert><Einheit>` und beschreiben die maximal vom Dogu nutzbare Menge an Speicher.
Zu beachten ist hier, dass zwischen dem numerischen Wert und der Einheit kein Leerzeichen stehen darf.
Verfügbare Einheiten sind `b`, `k`, `m` und `g` (für byte, kibibyte, mebibyte und gibibyte).

Das Setzen der Werte kann über folgende Wege erfolgen:
- `doguctl config container_config/memory_limit 1g`
- `cesapp edit-config <doguname>` (nur vom Host aus)
- `etcdctl set /config/<doguname>/container_config/memory_limit "1g"` (nur vom Host aus)

Um die Limitierungen zu übernehmen, muss das Dogu neu erstellt (`cesapp recreate <doguname>`) und anschließend neu gestartet (`cesapp start <doguname>`) werden.

#### Überprüfen der Limitierung

Die Limitierung des Speichers (nur RAM, kein Swap) kann mithilfe von `docker stats <doguname>` überprüft werden.
Die Spalte `MEM USAGE / LIMIT` sollte das gesetzte Speicherlimit korrekt anzeigen.

#### Limitierung in Java-Dogus

Ein Sonderfall stellt die Limitierung eines Java-Prozesses dar. Enthält ein Dogu einen Java-Prozess, können folgende zusätzliche Einträge in die `dogu.json` eingebaut werden:

```json
{
  "Configuration": [
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
  ]
}
```

**Hinweise:**
- Die Prozente beziehen sich immer auf den limitierten Speicher eines Containers. Sind keine Limitierung gesetzt, werden die Prozente ebenfalls ignoriert.
- Ein Java-Prozess sollte nicht einen zu großen Anteil des Speichers zugewiesen bekommen. Man sollte immer etwaige andere Prozesse beachten, die ebenfalls Speicher benötigen. Falls dies nicht passiert, kann es zu Abstürzen des eigentlichen Programmes kommen.
- Die so konfigurierten Werte müssen in den Start-Skripten des Dogus dem entsprechenden Java-Prozess als Parameter mitgegeben werden. Eine Referenzimplementierung findet sich im [Nexus-Dogu](https://github.com/cloudogu/nexus/blob/77bdcfdbe0787c85d2d9b168dc38ff04b225706d/resources/util.sh#L52).

### Backup & Restore-Fähigkeit

Ein Feature des Cloudogu EcoSystems ist, dass Dogus sich über ein zentrales Backupsystem sichern und wiederherstellen lassen. Damit das funktioniert, müssen alle Dogus ihre veränderlichen Daten in Volumes auslagern. Dieses Vorgehen wird bereits im [Compendium](../core/compendium_de.md#volumes) beschrieben.

Bei der Entwicklung eines Dogus ist darauf zu achten, dass das Dogu nach einem erfolgreichen Backup- und Restore-Vorgang weiterhin normal funktioniert und alle Daten und Funktionen vorhanden sind. Dazu müssen zuerst alle Volumes in der `dogu.json`, die Produktivdaten enthalten, mit dem [NeedsBackup](../core/compendium_de.md#needsbackup)-Flag gekennzeichnet werden. Danach sollte das Dogu gebaut und mit Testdaten gefüllt werden.

Anschließend führt man ein Backup und ein Restore des Systems durch. Dazu können die Befehle [cesapp backup](https://docs.cloudogu.com/de/docs/system-components/cesapp/operations/backup/) und [cesapp restore](https://docs.cloudogu.com/de/docs/system-components/cesapp/operations/restore/) genutzt werden.

Sind alle Dogus wieder hochgefahren, testet man, ob das eigene Dogu normal läuft und alle Testdaten weiterhin vorhanden sind.

### Änderbarkeit der Admin-Gruppe

Bei der Installation des Cloudogu EcoSystems wird eine globale Admin-Gruppe eingestellt. Alle Mitglieder dieser Gruppe sollen in allen Dogus Admin-Rechte bekommen. Dies muss bei der Entwicklung des Dogus sichergestellt werden. Der Name der global definierten Admin-Gruppe lässt sich aus dem Dogu heraus mit `doguctl` abfragen: `doguctl config --global 'admin_group'`

Zusätzlich kann die Admin-Gruppe im Nachhinein geändert werden. Das Dogu muss dann (ggf. nach einem Neustart) so reagieren, dass nur noch die Mitglieder der neuen Admin-Gruppe Admin-Rechte erhalten. Nutzer in der alten Admin-Gruppe erhalten nur noch normale Nutzerrechte, sofern sie sich nicht auch in der neuen Admin-Gruppe befinden.

### Änderbarkeit der FQDN

Die FQDN des Cloudogu EcoSystems ist global in der Registry gespeichert und kann von den Dogus mittels [`doguctl`](#die-nutzung-von-doguctl) ausgelesen werden. Falls nötig, kann sie in die Dogu-Konfiguration integriert werden (siehe auch [Erstellung einer exemplarischen Anwendung](../core/basics_de.md#5-erstellung-einer-exemplarischen-anwendung). Zusätzlich muss bei der Dogu-Entwicklung darauf geachtet werden, dass die FQDN änderbar ist. Das Dogu sollte also in der Lage sein (ggf. nach einem Neustart), eine neue FQDN auszulesen und seine Konfiguration an diese neue FQDN anzupassen.

### Logging-Verhalten steuern

Dieser Abschnitt zeigt, wie eine einheitliche, abstrahierte Einstellmöglichkeit geschaffen wird, mit der man das Log-Verhalten in allen Dogus auf die gleiche Art und Weise einstellen kann. Dazu werden vier verschiedene über die Registry setzbare Log-Level genutzt: `ERROR`, `WARN`, `INFO` und `DEBUG`.

Mit diesen abstrakten Log-Leveln soll der Administrator unterstützt werden. Er muss nicht wissen, wie ein Dogu-spezifisches Log-Level heißt, sondern kann stattdessen auf vier Grundwerte zurückgreifen und diese gefahrlos in allen Dogus wiederverwenden.

Dogu-Entwickler müssen darauf achten, dass diese vier Log-Level sinnvoll auf die Log-Level der Software im Dogu gemappt werden, welche ggf. andere Namen haben können (bspw. `TRACE` oder `FATAL`).

#### Hintergründe zum Log-Level

Niemand liest Logs. Wenn sie jemand liest, dann nicht gerne, und zwar aus zwei Gründen:
1. Standard-Logs enthalten für den Betrachter überwiegend unwichtige Information
2. Logs werden nur dann gelesen, wenn ein Problem auftritt

Die Log-Level-Steuerung begegnet diesen Umständen.

Wenn ein Dogu standardmäßig und dauerhaft zu viele Zeilen loggt, dann wird Administrierenden oder Supportleistenden das Leben schwerer gemacht als nötig.
Denn sie müssen sich durch einen Berg Zeilen arbeiten und die Logs in relevante und irrelevante Informationen trennen.
Zudem kann es passieren, dass bei übertriebenen Logging (z. B. lange Stacktraces verbunden mit Container Crash Loops) zu so viel Daten führen kann, dass die Festplatte voll läuft und keine Log-Rotation mehr möglich ist.

#### Log-Level und deren Spielräume

Die Log-Level sind nach Detailgrad von grob nach fein sortiert und enthalten den jeweils darüber liegenden. Bspw. enthalten Log-Ausgaben des Levels `WARN` also auch Log-Ausgaben der Levels `ERROR`.

Der Entwickler hat damit starke Freiheiten, welche Logs bei welchem Log-Level ausgegeben werden.

Um zwischen relevant und irrelevant zu unterscheiden, gibt es natürlich Interpretationsspielraum.
Cloudogu als Dogu-Anbieter unterstützt den Kunden darin, für seine Produktivsysteme das Logging so gering und so prägnant wie möglich zu gestalten.
Das bedeutet, aufgrund unserer Erfahrung sowohl als Softwareentwicklende als auch als Dogu-Benutzende entscheiden wir für den Kunden,
welche Log-Quellen, Log-Zeilen usw. je Dogu und Log-Level angemessen sind.

##### `ERROR`

Log-Level analog zu `ERROR` finden sich sehr häufig auf Produktionssystemen. Sie werden genutzt, damit der Betrieb oder Support keine überflüssigen Zusatzinformationen durchgehen muss, sondern effizient zum Kern von Problemen vorstoßen kann.

Logausgaben dieses Levels sollten nur Ausgaben enthalten, die auf akute Fehler, Betriebsverhinderung oder Sicherheitsverstöße usw. hinweisen.

Beispiele:
- Reparatur korrupter Daten in einer Datenbank
- Applikationsfehler
- ungeplanter Stopp oder Restart von Diensten oder Applikationen
- akute Konnektivitätsfehler
    - z. B. Postgresql-Dogu ist auch nach mehreren Minuten nicht erreichbar
- _wiederholte_, fehlgeschlagene Login-Versuche
- akut drohender oder akut bestehender Mangel an Ressourcen
- Meldungen mit `FATAL`- oder `SEVERE`-Log-Level

##### `WARN`

Log-Level analog zu `WARN` finden sich häufig auf Produktions- oder Staging-Systemen.

Logausgaben dieses Levels sind sehr ähnlich zu denen von `ERROR`. Im Unterschied zu diesen enthalten sie jedoch Ausgaben, die auf bevorstehende Fehler hinweisen.

Beispiele:
- fehlerhafte oder optimierbare Konfiguration wurde erkannt
- Aktualisierung von Applikation oder Bibliotheken werden vorgeschlagen
- mittelfristig drohender Mangel an Ressourcen
- singuläre Konnektivitätsfehler, die zeitlich begrenzt sind
    - z. B. Postgresql-Dogu ist beim ersten Versuch nicht erreichbar, doch nach wenigen Minuten schon
- _einzelne_, fehlgeschlagene Login-Versuche
- geplanter Start und Stopp von Diensten oder Applikationen

##### `INFO`

Ausgaben, die in besonderen, aber unkritischen Situationen von Interesse sein könnten. Ein _Catch-All_ für all die Ausgaben, die nicht wichtig genug für `WARN` sind, aber noch nicht tief in die Details der Anwendung einsteigen.

Beispiele:
- ein Subsystem wurde vollständig geladen
- Fehlerloses Ergebnis von Hintergrundjobs
- Erfolgreicher Start/Stop einer Komponente

##### `DEBUG`

Ausgaben, die in kritischen Situationen von Interesse sein könnten und/oder zur Fehlerfindung dienen.

Beispiele:
- eine relevante Entscheidung für eine Konfiguration wurde gefällt
- Meldungen mit `TRACE`-Log-Level

#### Mapping je Dogu

Diese vier abstrakten Log-Levels haben hin und wieder keine direkte Beziehung zu konkreten Log-Levels in den Tools oder Plugins im Dogu und müssen in der Dogu-Entwicklung manuell abgebildet werden. Der Grund dafür liegt darin, dass sich sowohl von den Tools als auch damit einhergehenden Plugins und Frameworks die Log-Levels zu stark in Quantität und Qualität unterscheiden.

Die Abbildung eines abstrakten Log-Levels liegt damit in der Hand der Dogu-Entwicklung. Hier kann man beispielsweise entscheiden, dass man das `TRACE`-Log-Level aus der Software auf das `DEBUG`-Log-Level des Dogus mappen möchte.

Diese Abbildung von abstrakten zu konkreten Log-Levels birgt den Vorteil der Flexibilität, z. B. wenn mehrere unterschiedliche Konfigurationsdateien mit unterschiedlichen Log-Leveln gefüllt werden müssen.

#### Wertebereich und Default

Zum Einstellen des Log-Levels wird je Dogu ein optionaler Key `/config/${dogu}/logging/root` eingeführt, der einen der folgenden abstrakten Log-Level-Werte annehmen kann.

- `ERROR`
- `WARN` (Defaultwert)
- `INFO`
- `DEBUG`

Wenn kein Key `/config/${dogu}/logging/root` existiert, dann verwendet `doguctl` automatisch den Wert `WARN`, wenn dieser ordnungsgemäß [als Default-Wert](#validierung-und-default-werte) konfiguriert wurde.

#### Validierung und Default-Werte

`doguctl` unterstützt Dogu-Entwickelnde darin, robust einen Log-Level für ein Dogu zu definieren. So bleibt lediglich die Abbildung eines gefundenen Log-Levels auf das jeweilige Logging-Framework im Dogu übrig.

In der `dogu.json` lässt sich ein validierbarer Log-Level einstellen, der bereits Dogu-seitig einen Standardwert mitbringt.

```json
{
  "Configuration": [
    {
      "Name": "logging/root",
      "Description": "Set the root log level to one of ERROR, WARN, INFO, DEBUG.",
      "Optional": true,
      "Default": "WARN",
      "Validation": {
        "Type": "ONE_OF",
        "Values": [
          "WARN",
          "DEBUG",
          "INFO",
          "ERROR"
        ]
      }
    }
  ]
}
```

Dieser Wert lässt sich mit `doguctl validate` überprüfen:

```
if [[ ! doguctl validate logging/root --silent ]]; then
    echo "WARNING: Found invalid value in logging/root. Resetting it to WARN"
    doguctl config logging/root WARN
fi
```

Der eingestellte bzw. Default-Wert aus der `dogu.json` lässt sich wie gewohnt abrufen. `doguctl` kümmert sich um die Gewinnung eines Default-Werts, wenn kein Wert gesetzt wurde:

```
rootLogLevel=$(doguctl config logging/root)
```

#### Erweiterbarkeit

Sollte der Bedarf für weitere, Dogu-spezifische Log-Levels bestehen, können diese mit zusätzlichen Schlüsseln erweitert werden (ebenfalls unter `/config/<doguname>/logging/`).
Solche Log-Levels sind von ihrer Natur sehr spezifisch.
Daher können andere Regeln für die Verwendung von solch spezifischen Keys gelten.
