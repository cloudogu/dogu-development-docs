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

## Chart veröffentlichen

Vor der Veröffentlichung müssen der Ziel-Namespace und die Zugangsdaten mit Cloudogu abgestimmt werden. Partner erhalten dafür einen eigenen Namespace in `registry.cloudogu.com`.

Dieses Beispiel verwendet öffentliche Images von Docker Hub. Deshalb müssen keine Images in unsere Registry gepusht werden. Enthält ein Dogu nicht-öffentliche Images, werden sie unter diesem Pfad abgelegt:

```text
registry.cloudogu.com/<namespace>/dogu/v3/images/<image>:<tag>
```

An der Cloudogu-Registry anmelden:

```shell
helm registry login registry.cloudogu.com
```

Danach das bereits paketierte Chart pushen:

```shell
helm push dist/xwiki-1.7.1-1.tgz \
  oci://registry.cloudogu.com/<namespace>/dogu/v3/charts
```

`<namespace>` durch den vereinbarten Partner-Namespace ersetzen. Zugangsdaten gehören nicht in das Chart oder in das Git-Repository.

Der Partner-Namespace wird in der Dogu-Registry dem OCI-Chart-Repository zugeordnet. Dadurch kann das veröffentlichte Dogu über die Dogu-Registry gefunden werden.
