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

Das Dogu aktualisieren:

```shell
cldk dogu dev-push
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

Da das EcoSystem auch in Air-Gapped-Umgebungen lauffähig sein soll, müssen in der `mirror-patch-tpl.yaml` Image-Referenzen und deren Nutzung in der `values.yaml` gepflegt werden. Diese können dann gespiegelt, und die Image-Referenzen in der `values.yaml` angepasst werden.

Die Image-Spiegelung erfolgt vor der Installation. Eine vorhandene `runtime-patch-tpl.yaml` bleibt dabei unverändert; sie wird erst vom Dogu-Operator ausgewertet.

Das Chart verwendet ein XWiki-Image und ein MySQL-Image. Das MySQL-Image läuft auch im Init-Container von XWiki, der auf die Datenbank wartet. Das XWiki-Chart setzt dessen Adresse ohne Registry zusammen, deshalb wird sie im Patch vollständig gesetzt.
Deswegen sieht die `mirror-patch-tpl.yaml` wie folgt aus und wird im selben Verzeichnis wie die `Chart.yaml` abgelegt:

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
      initContainers:
        database:
          # Vollständige Adresse für den Init-Container, der auf die Datenbank wartet
          image: "{{ .images.mysql }}"
```

Die Patches für die `values.yaml` werden dabei als Go-Templates angegeben.
Dabei werden für das Verarbeiten von Image-Referenzen folgende zusätzliche Funktionen zur Verfügung gestellt:

- `registryFrom` liefert die Registry, zum Beispiel `docker.io`.
- `repositoryFrom` liefert das Repository, zum Beispiel `library/xwiki`.
- `tagFrom` liefert den Tag, zum Beispiel `18.6.0-mysql-tomcat`.

## CES-Konfiguration über Runtime-Patches einbinden

Neben den Mirror-Patches für Image-Adressen gibt es Runtime-Patches für die globale CES-Konfiguration.

Mit Runtime-Patches kann der Dogu-Operator globale CES-Konfiguration in Chart-Values einsetzen, ohne die Templates eines Upstream-Charts zu verändern.

Die Verarbeitung erfolgt in drei Schritten:

1. Der Operator liest die globale CES-Konfiguration, wertet die Runtime-Patches aus und führt die Values-Quellen zusammen.
2. Die Values werden gegen das vorhandene Chart-Schema geprüft, sofern die Schema-Validierung nicht deaktiviert ist. Anschließend wird das Chart zur Vorabprüfung gerendert.
3. Bei erfolgreicher Prüfung werden die Values in ein HelmRelease geschrieben. Der Helm-Controller führt mit Helm die Installation oder Aktualisierung im Cluster aus.

Die `runtime-patch-tpl.yaml` wird neben `Chart.yaml` abgelegt. Sie enthält direkt die gewünschten Chart-Values.

`globalConfig` ist eine Template-Hilfsfunktion. Sie liest Werte aus der ConfigMap `global-config` im Namespace des Dogus und setzt sie in die Runtime-Values ein.

### Einen Runtime-Patch ergänzen

Die Schlüssel müssen zur `values.yaml` des Charts passen. Werte für ein Subchart stehen unter dessen Namen oder Alias (hier `backend`). Verschachtelte Maps und Listen werden unterstützt.

```yaml
application:
  host: '{{ globalConfig "fqdn" }}'
  certificate: '{{ globalConfig "certificate/server.crt" }}'
backend:
  applicationHosts:
    - '{{ globalConfig "fqdn" }}'
```

Die Datei muss schon vor der Auswertung durch den Operator gültiges YAML sein. Template-Ausdrücke werden deshalb in Anführungszeichen gesetzt. Der Operator wertet nur String-Werte aus, auch innerhalb von Listen. Schlüssel und Struktur bleiben unverändert.

Die Datei muss genau ein YAML-Dokument mit einer Map auf oberster Ebene enthalten. Wenn keine Runtime-Values benötigt werden, wird die Datei weggelassen. Der Inhalt `{}` ist als leere Map ebenfalls gültig. Eine leere Datei, eine Datei nur mit Leerraum oder mit dem Inhalt `null` ist nicht erlaubt.

### Konfiguration lesen

- `/` trennt Pfadsegmente; Punkte bleiben Teil eines Schlüssels. `certificate/server.crt` liest beispielsweise das Zertifikat unter diesem Pfad.
- Der angefragte Wert muss ein String sein; fehlende Schlüssel oder andere Datentypen führen zu einem Fehler. Ein leerer String ist erlaubt.
- Eingesetzte Werte bleiben Strings. Anführungszeichen und Zeilenumbrüche, etwa in Zertifikaten, bleiben erhalten.

`globalConfig` ist die einzige zusätzlich bereitgestellte Template-Funktion.

### Templates für die Anwendung erhalten

Auch in mehrzeiligen Strings (`|`) wertet der Operator `{{ ... }}` aus. Ausdrücke, die erst die Anwendung auswerten soll, werden als String ausgegeben:

```yaml
logConfiguration: |
  date: {{ "{{ .Date }}" }}
  line: {{ "{{ .LogLineJSON }}" }}
```

Die Anwendung erhält `date: {{ .Date }}` und `line: {{ .LogLineJSON }}`. Der Operator wertet das Ergebnis nicht erneut aus. Anführungszeichen oder `|` allein verhindern die Template-Auswertung nicht. Statische Anwendungsvorlagen ohne CES-Konfiguration können direkt in `values.yaml` bleiben.

### Priorität und Konfigurationsänderungen

Runtime-Values überschreiben die Chart-Defaults und `global.cesConfig`. Sie werden ihrerseits von `spec.values` und `spec.mappedValues` der Dogu-CR überschrieben. `globalConfig` liest immer die globale Konfiguration, nicht die durch `spec.values` überschriebenen Werte.

Ändert sich `global-config`, wertet der Operator die Runtime-Patches erneut aus und rollt die neuen Values aus, ohne Pods neu zu starten. Soll ein Workload bei geänderten ConfigMaps oder Secrets neu starten, kann er z. B. mit `k8s.cloudogu.com/reload-auto: "true"` annotiert werden.

Die vollständige Priorisierung, das Zusammenführen der Values und das Verhalten bei Konfigurationsänderungen beschreibt [v3-Dogus konfigurieren](https://github.com/cloudogu/k8s-dogu-operator/blob/develop/docs/operations/v3/configuring_v3_dogus_de.md).

### Runtime-Patches testen

Runtime-Patches wertet nur der Dogu-Operator aus, `helm template` oder `helm install` dagegen nicht. Zum Testen das Dogu mit `cldk dogu dev-push` in die lokale Umgebung bringen und die Values prüfen, die der Operator in das HelmRelease geschrieben hat:

```shell
cldk dogu dev-push
helm get values xwiki --namespace ecosystem
```

## Nächster Schritt

Das Chart kann jetzt weitergegeben werden. [Ein Dogu V3 releasen](../guides/release_de.md) erklärt, wie der Registry-Zugang beantragt, das Dogu mit `cldk dogu release` veröffentlicht und das Release geprüft wird.
