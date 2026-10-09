# XWiki in das CES integrieren

Diese Anleitung ergänzt [Vom Helm-Chart zum Dogu V3](quickstart_de.md). Sie fügt XWiki zum Warp-Menü hinzu, erklärt Voraussetzungen für die zentrale CES-Anmeldung und bereitet das Chart für die Weitergabe vor.

## XWiki zum Warp-Menü hinzufügen

Ein `WarpMenuEntry` fügt XWiki zur zentralen Navigation des CES hinzu. Im Verzeichnis `templates` die Datei `warp-menu-entry.yaml` erstellen:

```yaml
apiVersion: k8s.cloudogu.com/v1
kind: WarpMenuEntry
metadata:
  name: {{ .Release.Name }}
  labels:
    app.kubernetes.io/name: xwiki
    app.kubernetes.io/instance: {{ .Release.Name }}
spec:
  category: Documentation
  displayName:
    de: XWiki
    en: XWiki
  path: /xwiki
```

Das Chart aktualisieren:

```shell
helm upgrade --install xwiki . \
  --namespace ecosystem \
  --wait \
  --timeout 15m
```

Auf den fertigen Eintrag warten und das Ergebnis prüfen:

```shell
kubectl wait \
  --for=jsonpath='{.status.conditions[?(@.type=="Ready")].status}'=True \
  warpmenuentry/xwiki \
  --namespace ecosystem \
  --timeout 2m

kubectl get warpmenuentry xwiki --namespace ecosystem --output wide
```

Nach dem Neuladen der CES-Seite erscheint XWiki im Warp-Menü.

## Anmeldung und Berechtigungen

Eine `AuthRegistration` registriert XWiki beim Identity Provider des CES (CAS).
XWiki benötigt zusätzlich einen passenden Authenticator, um die Anmeldung über CAS zu verwenden.

Die Einrichtung des Authenticators sowie die Übernahme von LDAP-Gruppen und XWiki-Rechten gehören noch nicht zu dieser Anleitung. Diese Punkte hängen von der jeweiligen Anwendung ab und müssen vor der Veröffentlichung des Dogus festgelegt werden.

Hier muss sichergestellt werden, dass die in der Globalen Config hinterlegte Admin-Gruppe in das Dogu synchronisiert wird und dort Admin-Berechtigungen erhält. Außerdem muss das Dogu auch auf Änderungen der Admin-Gruppe reagieren bzw. diese spätestens nach einem Neustart synchronisieren.

Weitere Informationen:

- [Multinode-Umgebung und Dogu-V3-Ressourcen](../concepts/multinode-environment_de.md)
- [XWiki OpenID Connect Authenticator](https://github.com/xwiki-contrib/oidc/tree/master/oidc-authenticator)
- [CAS als OpenID-Connect-Anbieter](https://apereo.github.io/cas/development/authentication/OIDC-Authentication.html)

Grundsätzlich könnte die `AuthRegistration` für XWiki wie folgt aussehen:

```yaml
apiVersion: k8s.cloudogu.com/v1
kind: AuthRegistration
metadata:
  name: {{ .Release.Name }}
spec:
  consumer: {{ .Release.Name }}
  protocol: OIDC
  secretRef: xwiki-oidc-credentials
```

Der Auth-Registration-Operator registriert XWiki bei CAS. Danach schreibt er Client-ID, Client-Secret und Issuer-URL in das Secret `xwiki-oidc-credentials`.

## Images für die spätere Spiegelung vorbereiten

Da das EcoSystem auch in Air-Gapped-Umgebungen lauffähig sein soll, müssen in der `chart-patch-tpl.yaml` Image-Referenzen und deren Nutzung in der `values.yaml` gepflegt werden. Diese können dann gespiegelt, und die Image-Referenzen in der `values.yaml` angepasst werden.

Das Chart verwendet ein XWiki-Image und ein MySQL-Image.
Deswegen sieht die `chart-patch-tpl.yaml` wie folgt aus und wird im selben Verzeichnis wie die `Chart.yaml` abgelegt:

```yaml
apiVersion: v1
values:
  # Alle Container-Images des Charts
  images:
    xwiki: "docker.io/library/xwiki:18.6.0-mysql-tomcat"
    mysql: "docker.io/bitnamilegacy/mysql:8.4.4-debian-12-r7"

patches:
  # Image-Adressen in values.yaml für die Ziel-Registry anpassen
  values.yaml:
    xwiki:
      image:
        # Registry und Repository des XWiki-Images übernehmen
        name: "{{ registryFrom .images.xwiki }}/{{ repositoryFrom .images.xwiki }}"
        tag: "{{ tagFrom .images.xwiki }}"
      mysql:
        image:
          # Das MySQL-Chart speichert Registry und Repository getrennt
          registry: "{{ registryFrom .images.mysql }}"
          repository: "{{ repositoryFrom .images.mysql }}"
          tag: "{{ tagFrom .images.mysql }}"
```

Die Patches für die `values.yaml` werden dabei als Go-Templates angegeben.
Dabei werden für das Verarbeiten von Image-Referenzen folgende zusätzliche Funktionen zur Verfügung gestellt:

- `registryFrom` liefert die Registry, zum Beispiel `docker.io`.
- `repositoryFrom` liefert das Repository, zum Beispiel `library/xwiki`.
- `tagFrom` liefert den Tag, zum Beispiel `18.6.0-mysql-tomcat`.

## CES-Konfiguration über Runtime-Patches einbinden

Mit Runtime-Patches kann der Dogu-Operator globale CES-Konfiguration in Chart-Values einsetzen, ohne die Templates eines Upstream-Charts zu verändern.

Ergänze in `chart-patch-tpl.yaml` einen Abschnitt `runtimePatches` auf oberster Ebene. Die Datei liegt neben `Chart.yaml`. Behalte die bestehenden Abschnitte `values` und `patches` für die Image-Spiegelung bei.

### Einen Runtime-Patch ergänzen

Dieses Beispiel lässt die Image-Abschnitte aus und verwendet beispielhafte Chart-Values. Passe die Zielschlüssel an die `values.yaml` deines Charts an. Für Abhängigkeiten stehen die Overrides unter dem Namen oder Alias des Subcharts (hier `backend`).

```yaml
apiVersion: v1
runtimePatches:
  values.yaml:
    {{ if empty (globalConfig "fqdn") }}{{ fail "CES fqdn muss gesetzt sein" }}{{ end }}
    application:
      host: {{ globalConfig "fqdn" | quote }}
      proxyHost: {{ globalConfig "proxy/host" | default "" | quote }}
      certificate: {{ globalConfig "certificates/server.crt" | default "" | quote }}
      certificates:
        {{ globalConfig "certificates" | default dict | toYaml | nindent 8 }}
    backend:
      applicationHost: {{ globalConfig "fqdn" | quote }}
```

Als Runtime-Patch-Ziel wird derzeit nur `values.yaml` unterstützt. Behalte die gezeigte Map-Struktur bei, statt einen `|`-Block-String zu verwenden. Nutze keine YAML-Anker oder mehrere YAML-Dokumente. Eine fehlende Datei oder `runtimePatches: {}` ist erlaubt; eine vorhandene leere, nur aus Kommentaren bestehende oder ungültige Datei führt zu einem Fehler.

### Konfiguration lesen und Werte formatieren

- `globalConfig` liest `config.yaml` aus der ConfigMap `global-config` im Namespace des Dogus. `/` trennt Pfadsegmente; Punkte bleiben Teil eines Schlüssels. `certificates/server.crt` liest einen Wert, `certificates` den gesamten Unterbaum.
- Fehlende Pfade liefern `nil`. Verwende `default` für optionale Werte.
- `default` ersetzt auch leere Strings, leere Maps/Listen, `false` und die Zahl `0`.
- `quote` erzeugt einen sicher quotierten YAML-String, auch für mehrzeilige Zertifikate.
- `toYaml | nindent 8` fügt hier einen Unterbaum mit acht Leerzeichen Einrückung ein. Passe die Zahl bei anderer Verschachtelung an.
- `fail` bricht die Auswertung mit deiner Fehlermeldung ab. Im Beispiel wird damit eine nicht leere `fqdn` verlangt.

Die [Sprig-Funktionen](https://masterminds.github.io/sprig/) sind verfügbar, außer `env`, `expandenv` und `getHostByName`.

Runtime-Patches sind keine Helm-Templates: `.Values`, `.Release`, `.images`, Helm-Hilfsfunktionen wie `include`, `tpl`, `lookup` und `required` sowie die Image-Hilfsfunktionen sind nicht verfügbar. `globalConfig` steht nur in Runtime-Patches zur Verfügung.

### Overrides und Konfigurationsänderungen

Die Priorität steigt von links nach rechts:

`Chart-Defaults → global.cesConfig → runtimePatches → spec.values → spec.mappedValues`

Bei Änderungen an `global-config` werden die Runtime-Patches erneut ausgewertet. Ob die Anwendung die Konfiguration neu lädt oder einen Neustart benötigt, hängt vom Chart und der Anwendung ab.

**Testen:** Verwende ein vom Operator verwaltetes Dogu in einem CES mit aktivierter V3-Verarbeitung. Ein normales `helm template` oder `helm install` wertet Runtime-Patches nicht aus – auch die Helm-Befehle im weiteren Verlauf dieser Anleitung nicht.

## Chart in die lokale Registry pushen

Die lokale Registry wurde zusammen mit der Quickstart-Umgebung eingerichtet. Sie ist über `localhost:5001` erreichbar.

Zuerst die Abhängigkeiten laden und das Chart paketieren:

```shell
helm dependency build .
helm lint .

mkdir -p dist
helm package . --destination dist
```

Das erzeugte Paket in die lokale OCI-Registry pushen:

```shell
helm push dist/xwiki-1.7.1-1.tgz \
  oci://localhost:5001/quickstart/charts \
  --plain-http
```

Das Chart kann nun auch direkt aus der lokalen Registry installiert werden:

```shell
helm upgrade --install xwiki \
  oci://localhost:5001/quickstart/charts/xwiki \
  --version 1.7.1-1 \
  --plain-http \
  --namespace ecosystem \
  --wait \
  --timeout 15m
```

## Nächster Schritt

Das Chart kann jetzt weitergegeben werden. [Ein Dogu V3 releasen](../guides/release_de.md) erklärt, wie der Registry-Zugang beantragt, das Chart veröffentlicht und das Release geprüft wird.
