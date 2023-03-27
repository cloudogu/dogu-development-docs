# Relevante Funktionalitäten
Dieses Kapitel beschreibt die Features und mögliche Implementierungsideen/-Lösungen derer Funktionalitäten, die überhaupt ein echtes Dogu ausmachen und wiederkehrende Probleme lösen. 

Die folgenden Abschnitte beschäftigen sich daher mit wiederkehrenden Funktionen, wie ein Dogu sich in die Landschaft des Cloudogu EcoSystem einbetten kann.

## Authentifizierung
Lorem ipsum Einführungstext

### CAS-Protokoll
Authentifizierung innerhalb des Cloudogu EcoSystem geschieht über das [CAS-Protokoll](https://apereo.github.io/cas/6.5.x/protocol/CAS-Protocol.html), das Single Sign-on (SSO) und Single Logout (SLO) ermöglicht.
Folgendes Diagramm zeigt die Beteiligten an der Authentifizierung.
Bevor ein Dogu (hier am Beispiel von Redmine) an diesem Prozess teilnehmen kann, muss das Dogu intern einen Satz von CAS-URLs (rote Pfeile) konfigurieren:
CAS log-in URL (zur Weiterleitung von Benutzenden an das Webformular)
CAS validation URL (zur Validierung von Service Tickets)
CAS log-out URL (zur Invalidierung einer SSO-Session)
Die Service-Erkennung in CAS gegenüber des Dogus (blauer Pfeil) geschieht automatisch während der Dogu-Installation und findet im Folgenden keine weitere Betrachtung.

Die tatsächliche Authentifizierung geschieht über eine Abfolge von HTTP-Redirects und den Austausch von Session-Cookies im Hintergrund, von denen Benutzende nichts wahrnehmen. Die Anzeige des CAS-Login-Formulars als aktiver Anmeldeschritt sticht dabei stark heraus.
Eine grobe Übersicht über den Anmeldeprozess mit den Beteiligten bietet die folgende Abbildung.
Das SSO des CAS reduziert diesen Prozess bei der Anmeldung bei weiteren Dogus deutlich.

Weitere Informationen und eine genauere Abbildung vor, während und nach einer Authentifizierung bietet die CAS-Dokumentation.

### OAuth-Protokoll

CAS bietet OAuth/OIDC als Protokoll zur Authentifizierung samt SSO/SSL an.
Im Folgenden werden die Spezifikation des OAuth Protokolls in CAS beschrieben.

#### OAuth Service Account für Dogu erstellen

Damit ein Dogu die OAuth/OIDC-Endpunkte des CAS benutzen kann, muss sich dieser beim CAS als Client anmelden.
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

### OAuth Endpunkte und Ablauf

Die folgenden Schritte beschreiben einen erfolgreichen Ablauf der OAuth-Authentifizierung.

1. Anfordern eines Kurzzeit-Tokens: Siehe Abschnitt unten "OAuth-Authorize-Endpunkt"
2. Kurzzeittoken gegen ein Langzeittoken tauschen: Siehe Abschnitt unten "AccessToken-Endpunkt"
3. Langzeittoken kann nun zu Authentifizierung gegen Ressourcen benutzen werden.
   Derzeit bietet CAS nur das Profil der User als Resource an: Siehe Abschnitt unten "OAuth-Userprofil"

#### OAuth-Authorize-Endpunkt

Dieser Endpunkt dient als initialer Start der OAuth-Authorisation.
Der Authorisation-Endpunkt wird benutzt, um ein kurzlebiges Token vom CAS anzufordern.

**URL** : `<fqdn>/api/authorize`

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

**URL** : `<fqdn>/api/accessToken`

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

Dieser Endpunkt dient zur Abfrage des Userprofil vom eingeloggten User mithilfe eines Langzeittokens (`access_token`).

**URL** : `<fqdn>/api/profile`

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

### OIDC-Protokoll


## Auf die Registry zugreifen


## Aufbau und Best Practices von `startup.sh` 


## Service Accounts
Lorem ipsum Einführungstext

### Service Accounts produzieren 


### Service Accounts konsumieren

## Dogu-Upgrades


## Typische Dogu-Features
Lorem ipsum Einführungstext

### Memory-/Swap-Limit


### Backup & Restore-Fähigkeit


### Änderbarkeit der Admin-Gruppe


### Änderbarkeit der FQDN


### Loglevel mappen und ändern

