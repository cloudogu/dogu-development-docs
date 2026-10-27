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

Da das EcoSystem auch in Air-Gapped-Umgebungen lauffähig sein soll, müssen in der `mirror-patch-tpl.yaml` Image-Referenzen und deren Nutzung in der `values.yaml` gepflegt werden. Diese können dann gespiegelt, und die Image-Referenzen in der `values.yaml` angepasst werden.

Die Image-Spiegelung erfolgt vor der Installation. Eine vorhandene `runtime-patch-tpl.yaml` bleibt dabei unverändert; sie wird erst vom Dogu-Operator ausgewertet.

Das Chart verwendet ein XWiki-Image und ein MySQL-Image.
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
  certificate: '{{ globalConfig "certificates/server.crt" }}'
backend:
  applicationHosts:
    - '{{ globalConfig "fqdn" }}'
```

Die Datei muss schon vor der Auswertung durch den Operator gültiges YAML sein. Template-Ausdrücke werden deshalb in Anführungszeichen gesetzt. Der Operator wertet nur String-Werte aus, auch innerhalb von Listen. Schlüssel und Struktur bleiben unverändert.

Die Datei muss genau ein YAML-Dokument mit einer Map auf oberster Ebene enthalten. Wenn keine Runtime-Values benötigt werden, wird die Datei weggelassen. Der Inhalt `{}` ist als leere Map ebenfalls gültig. Eine leere Datei, eine Datei nur mit Leerraum oder mit dem Inhalt `null` ist nicht erlaubt.

### Konfiguration lesen

- `/` trennt Pfadsegmente; Punkte bleiben Teil eines Schlüssels. `certificates/server.crt` liest beispielsweise das Zertifikat unter diesem Pfad.
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

### Overrides und Konfigurationsänderungen

Die Priorität steigt von links nach rechts:

`Chart-Defaults → global.cesConfig → Runtime-Values → spec.values → spec.mappedValues`

`globalConfig` liest immer die globale Konfiguration, nicht die durch `spec.values` überschriebenen Werte.

Änderungen an `global-config` lösen einen erneuten Abgleich der V3-Dogus aus. Dabei werden die Runtime-Patches erneut ausgewertet und bei erfolgreicher Prüfung die HelmRelease-Values aktualisiert. Der Abgleich allein startet keine Pods neu.

Ob die Anwendung geänderte Konfiguration neu lädt oder ein Pod-Rollout nötig ist, hängt vom Chart und der Anwendung ab. Für geänderte ConfigMaps und Secrets kann ein aktivierter Reloader den Rollout auslösen, wenn der Workload beispielsweise mit `k8s.cloudogu.com/reload-auto: "true"` annotiert ist.

**Testen:** Dafür wird ein vom Operator verwaltetes Dogu in einem CES mit aktivierter V3-Verarbeitung benötigt. Ein normales `helm template` oder `helm install` wertet Runtime-Patches nicht aus – auch die Helm-Befehle im weiteren Verlauf dieser Anleitung nicht.

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
