# Troubleshooting

Das Kapitel Troubleshooting bietet eine Hilfestellung, mögliche Fehler während der Dogu-Entwicklung zu lokalisieren.
Für bekannte Fehler werden außerdem Lösungen dargestellt.

## Wie vermeide ich Fehler aufgrund von ungültigen Konfigurationseinträgen?

- Gültige Registry-Pfade sind zu beachten
- `doguctl` kann Konfigurationswerte validieren. Siehe [Validierung](relevant_functionalities_de.md#validierung-und-default-werte)
- Übersicht aller Keys: `etcdctl ls -r config/<doguname>`
- Aus dem Container ist es möglich, verschlüsselte Werte zu überprüfen:
  - `docker exec -it redmine doguctl config -e sa-postgresql/username`

## Das Dogu startet nicht und/oder befindet sich in einer Restart-Loop

Besteht eine ungefähre Vermutung zur Position der fehlerhaften Stelle, können zusätzliche `echo` - Ausgaben helfen, das Problem zu analysieren.

Falls weiterhin die Fehlerquelle unklar bleibt, ist es notwendig die Anweisungen innerhalb des Containers einzeln auszuführen:
- Shell in den Container: `docker exec -it <doguname> bash`
- Anschließend kann der Container debuggt werden:
  - Prüfung des Filesystems
  - Iterative Ausführung der Befehle in den Skripten, um den Fehler zu lokalisieren
  - Defekt mittels Bordmitteln im Container reparieren

Bei einer Restart-Loop haben sie folgende Möglichkeiten, den Container in einen stabilen Zustand zu bringen:
- Während der Entwicklung können sie am besten den Entrypoint des Containers überschreiben.
  - `ENTRYPOINT ["tail", "-f", "/dev/null"]` anstatt `CMD ["/resources/startup.sh"]`
  - Alternativ kann an beliebiger Stelle ein `sleep --infinite` in das Start-Skript geschrieben werden.
- In Produktion muss ein Container in einer Restart-Loop manuell gestoppt und mit einem Commit wieder neu gestartet werden:
```bash
docker stop ldap
# commit the stopped container to a local debug image
docker commit hashFromYourStoppedContainer_cebf8699cd9d debug/ldap
# the volume arguments can be generated with docker inspect
docker inspect ldap -f '{{ range.Mounts}}-v {{.Source}}:{{.Destination}} {{end}}'
# debug your dogu container
docker run -ti \
   --rm \
# here goes the docker volume list from above
   -v /etc/ces:/etc/ces \
   -v /var/lib/ces/ldap/volumes/_private:/private \
   -v /var/lib/ces/ldap/volumes/config:/etc/openldap/slapd.d \
   -v /var/lib/ces/ldap/volumes/db:/var/lib/openldap \
   --entrypoint=/bin/sh --network cesnet1 --name ldap --hostname ldap \ 
   debug/ldap
```

## Logging-Ausgaben fehlen oder haben eine falsche Granularität

Es gibt viele Gründe, warum es zu vollständig/teilweise fehlenden oder den falschen Logging-Ausgaben kommt.

- Wird ein falscher Pfad zum Auslesen des Log-Levels verwendet?
  - Der korrekte Pfad lautet: `config/<doguname>/logging/root`
- Wird ein ungültiges Log-Level verwendet?
  - Die vier validen Werte lauten: `ERROR`, `WARN`, `INFO`, `DEBUG`
- Wird der Wert beim Starten des Dogus nicht sinnvoll interpretiert?
   - Ein korrektes Beispiel wäre:
```bash
function mapDoguLogLevel() {
  local DEFAULT_LOGGING_KEY="logging/root"
  local LOG_LEVEL_ERROR="ERROR"
  local LOG_LEVEL_WARN="WARN"
  local LOG_LEVEL_INFO="INFO"
  local LOG_LEVEL_DEBUG="DEBUG"
  local DEFAULT_LOG_LEVEL=${LOG_LEVEL_WARN}
  
  echo "Mapping dogu specific log level"
  local currentLogLevel=$(doguctl config --default "${DEFAULT_LOG_LEVEL}" "${DEFAULT_LOGGING_KEY}")

  echo "${SCRIPT_LOG_PREFIX} Mapping ${currentLogLevel} to Catalina"
  case "${currentLogLevel}" in
    "${LOG_LEVEL_ERROR}")
      export CATALINA_LOGLEVEL="SEVERE"
    ;;
    "${LOG_LEVEL_INFO}")
      export CATALINA_LOGLEVEL="INFO"
    ;;
    "${LOG_LEVEL_DEBUG}")
      export CATALINA_LOGLEVEL="FINE"
    ;;
    *)
      echo "${SCRIPT_LOG_PREFIX} Falling back to WARNING"
      export CATALINA_LOGLEVEL="WARNING"
    ;;
  esac
}
```

## Volumes

Es ist ratsam, die Größe der Volumes regelmäßig zu überprüfen.
Diese werden unter dem Pfad `/var/lib/ces/<dogu>/volumes/` gemounted.

Ein Dogu sollte einen wachsenden Datenbestand ausschließlich in Docker Volumes speichern, die in der [`dogu.json`](https://github.com/cloudogu/dogu-development-docs/blob/main/docs/core/compendium_en.md#volumes) beschrieben werden. 

Volumes haben neben einer besseren Geschwindigkeit weitere Vorteile: Sie gehen nicht verloren, wenn der Container beendet wird. Außerdem lässt sich die Festplattenauslastung besser überwachen.

## Wo finde ich FQDN, Zertifikat und weitere allgemeine Konfigurationen?

siehe: [Registryeinträge](relevant_functionalities_de.md#weitere-registryeinträge)

## Wie werden Registryeinträge gelesen/geschrieben?

Mittels [doguctl](relevant_functionalities_de.md#die-nutzung-von-doguctl).
Es ist in unserem [Base-Image](https://github.com/cloudogu/base) enthalten.

## Wie kann ein Container neu instanziiert werden?

- `cesapp recreate <doguname>`

## Das Dogu ist nicht über Nginx erreichbar

- Wurde diese Umgebungsvariable im Dockerfile gesetzt?
  - `ENV SERVICE_TAGS=webapp`
- Konfiguration im Nginx-Container checken
  - `docker exec -it nginx bash`
  - `cat /etc/nginx/conf.d/app.conf`

## Dogu erscheint nicht im Warp Menü

- Für einen Eintrag im Warp-Menü muss das Tag `warp` und eine Kategorie in der dogu.json definiert werden.
- Das Dogu muss gültiges HTML mit einem abschließenden `<\html>`-Tag ausliefern.
- Das Dogu wurde wegen einer fehlerhaften Installation nicht registriert.

## Dogu-Volumes werden nicht von Backup & Restore beachtet

Damit Volume-Daten eines Dogus von dem Backup & Restore Mechanismus beachtet werden, muss das entsprechende Volume mit dem Feld `NeedsBackup` erweitert werden:

```json
{
  "Name": "db",
  "Path": "/var/lib/openldap",
  "NeedsBackup": true
}
```

## Freigeben von Speicherplatz

- Entfernung von nicht mehr benötigten Logs z.B. syslog
  - `> /var/log/syslog`
- Rotieren von Logs
  - `logrotate --force /etc/logrotate.conf`
- Entfernen von nicht mehr benötigten Docker-Images
  - `docker rmi my-image:1.2.3`

## Das Filesystem ist voll obwohl `df` dies nicht widerspiegelt

In diesem Fall kann es hilfreich sein, das Btrfs-Filesystem zu balancieren.

- `btrfs filesystem balance /var/lib/docker/btrfs`

**Achtung**:
- Der Prozess versetzt die Partition in einen read-only Modus, der bei einem Abbruch nicht zurückgesetzt wird.
In dem Fall ist ein Systemneustart notwendig.

## In welchen Verzeichnissen wird am meisten Speicherplatz verbraucht?

- `cd <dir> && du -d 1 -h .* | sort -h`

# Ubuntu Zeit neu synchronisieren

Bei längeren Offline-Zeiten des Cloudogu EcoSystem kann es notwendig sein, die Systemzeit neu zu synchronisieren:

- `systemctl restart systemd-timesyncd.service`

## Fehlerbehebung bei Netzwerkproblemen in einem Dogu

Dieses Kapitel gibt Ideen, wie man Netzwerkprobleme in einem Dogu untersuchen kann. Weiterführende Informationen zum Thema Dogu-Kommunikation bietet zudem das über [relevante Funktionalitäten](relevant_functionalities_de.md#unterstützte-kommunikationsprotokolle) an.

### Netzwerkverbindungen

Wenn unklar ist, welche Verbindungen gerade möglich sind, helfen übliche Methoden und Werkzeuge wie diese:

- Prüfen, ob der Container erfolgreich im Netzwerk angesiedelt ist
  - z. B. im Classic-CES: `docker inspect $doguName`
- Zugriffslogs des Containers prüfen
  - evtl. auf ein ausreichend eingestellte Log-Level achten
- Prüfen, ob der Netzwerkdienst auch tatsächlich läuft
  - häufig dauert es eine Weile, bis die Anwendung tatsächlich den Port öffnet und/oder aktiv darauf Netzwerkverbindungen annimmt
- ping
  - der schnelle Klassiker unter den Verbindungsprüfungen
  - allerdings verwendet Ping ein anderes Protokoll, als das was voraussichtlich verwendet werden soll
- netstat
  - Hilft bei der Frage, welche Verbindung zu welcher IP-Adresse und Port geöffnet ist.
  - im Container:
    ```bash
      while true; do \
        echo -n $(date '+%M:%S'); \
        echo -n "  " ; \
        netstat -an 2> /dev/null | grep $MY_EXPECTED_PORT ; \
        sleep 1; \
      done;
    ```
- nslookup/dig/`/etc/hosts`
  - Hilft bei der Frage, auf ob und auf welche IP-Adresse sich die FQDN oder ein Dogu auflösen lässt.
- wscat
  - Hilft bei der Frage, ob sich eine Websocket-Verbindung herstellen lässt. 
  ```bash
    wscat -n -L -c "wss://fqdn/deineUrl/"
    wscat -n -L -c "ws://dogu/deineDoguSpezifischeUrl/"
  ```

Wenn diese zuletzt genannten Werkzeute nicht im Container vorhanden sind, können sie ggf. über den jeweiligen Paketmanager (häufig `apk` oder `apt/apt-get`) nachträglich installiert werden, sofern eine Internetverbindung zu dem jeweiligen Paket-Repository möglich ist.

### Spezielle nginx-Routen

Bevor erklärt wird, wie spezielle nginx-Routen erzeugt werden können, muss die Erzeugung normaler nginx-Routen erklärt werden.
Dieser Abschnitt ist daher in zwei größere Teile unterteilt:

1. Wie werden normale nginx-Routen für Dogus erzeugt? 
2. Wie lassen sich nginx-Routen für Dogus anpassen?

#### Wie werden normale nginx-Routen für Dogus erzeugt?

Wenn ein Dogu installiert wird, dessen `Dockerfile` einen Container-Port mittels `EXPOSE` öffnet, dann erzeugt das Cloudogu EcoSystem automatisch ein nginx-URL-Rewrite für dieses Dogus.

Beispiel eines beliebigen Dogu-`Dockerfiles`. Die Datei [`dogu.json`](../core/compendium_de.md#type-dogu) spielt in diesem Beispiel keine Rolle, außer dass der Dogu-Name `my-dogu` lautet:

```Dockerfile
FROM registry.cloudogu.com/official/base:3.17.1-1
...
EXPOSE 8081
CMD ["/resources/startup.sh"]
```

Wenn das Dogu basierend auf diesem Dockerfile installiert wird, entsteht automatisch ein neuer nginx-Konfigurationseintrag. Das bedeutet, es kann grundsätzlich `https://my-ces-instance/my-dogu` als URL angewählt werden. <!-- markdown-link-check-disable-line -->

Die Qualität dieses Konfigurationseintrags hängt allerdings von dem [Health-Zustand](../core/compendium_de.md#healthchecks) des Dogus ab. Wenn das Dogu noch nicht _healthy_, also noch nicht betriebsbereit ist, wird dies erkannt und eine Meldung "Dogu is starting" mit dem HTTP-Status 503 wird ausgegeben. Die dazugehörige nginx-Konfiguration dazu etwa so aus:

```
server {
...
    location /my-dogu {                               
        error_page 503 /errors/starting.html;
        return 503     
    }
...
}
```

Wenn das Dogu jedoch vollumfänglich betriebsbereit und damit _healthy_ ist, wird dies erkannt und die dazugehörige nginx-Konfiguration so umgeschrieben, dass die Anfrage an den Dogu-Container weitergeleitet wird. Die dazugehörige nginx-Konfiguration dazu etwa so aus:

```
server {
...
    location /my-dogu {                               
        proxy_pass http://172.18.0.8:8081;     
    }
...
}
```

#### Wie lassen sich nginx-Routen für Dogus anpassen?

> [!CAUTION]
> Angepasste Routen können für Verwirrung sorgen. Sie sollten nur sparsam eingesetzt werden.

> [!CAUTION]
> Fehlerhafte nginx-Konfigurationseinträge können bei Überlappung ganze Teile des Cloudogu EcoSystems stören. 

Nun sollte klar sein, welches Routing erzeugt wird, wenn man in einem Dogu ein herkömmliches `Dockerfile` einsetzt.

##### Automatisierung durch Dockerfile-Umgebungsvariablen

Abhängig von der Applikation im Dogu, kann die erzeugte URL `https://my-ces-instance/my-dogu` vielleicht nicht ausreichen. In komplexeren Szenarien ist es unter Umständen nötig, weitere URLs zu erzeugen. <!-- markdown-link-check-disable-line -->

Die Eigenschaften der Dogu-Einträge in der nginx-Konfiguration lassen sich durch Umgebungsvariablen im `Dockerfile` anpassen und sogar um neue Einträge erweitern:

Quelle: [gliderlabs/registrator Docs](https://github.com/gliderlabs/registrator/blob/master/docs/user/services.md)

| Umgebungsvariable             | Wert                                                                            | Ergebnis                                               |
|-------------------------------|---------------------------------------------------------------------------------|--------------------------------------------------------|
| `SERVICE_TAGS`                | `webapp`                                                                        | ordnet dem Dogu eine neue URL zu                       |
| `SERVICE_NAME`                | `neuerServiceName`                                                              | ordnet dem Dogu eine neue URL zu                       |
| `SERVICE_<port>_NAME`         | `neuerServiceName`                                                              | ordnet nur für Port `<port>` dem Dogu eine neue URL zu |
| `SERVICE_<port>_TAGS`         | `webapp`                                                                        | ordnet nur für Port `<port>` dem Dogu Tags zu |
| `SERVICE_ADDITIONAL_SERVICES` | `[{"name": "eindeutiger-bezeichner", "location": "urlx", "pass": "/neue-url"}]` | erzeugt einen neuen `location`-Eintrag                 |
| `SERVICE_IGNORE`              | `true`                                                                          | erzeugt für das Dogu keine URL                         |
| `SERVICE_<port>_IGNORE`       | `true`                                                                          | erzeugt nur für Port `<port>` keine URL                |

In diesem Kontext ist die Umgebungsvariable `SERVICE_ADDITIONAL_SERVICES` interessant, denn damit lassen sich umfangreiche Anpassungen an dem neuen nginx-Service vornehmen. Angenommen das `Dockerfile` aus dem vorigen Abschnitt wird um einen zweiten `EXPOSE`-Eintrag erweitert, weil das Dogu nun zwei Ports bedienen kann:

Beispiel: Angepasstes `Dockerfile`:
```Dockerfile
FROM registry.cloudogu.com/official/base:3.17.1-1
ENV SERVICE_ADDITIONAL_SERVICES='[{"name": "my-dogu-speziallfall", "port:8088", "location": "alte-url", "pass": "/neue-url"}]'
...
EXPOSE 8081
EXPOSE 8088
CMD ["/resources/startup.sh"]
```

Beispiel einer resultierenden nginx-Konfiguration:
```
server {
...
    location /urlx {
        proxy_pass http://172.18.0.8:8088/neue-url;
    }
    location /my-dogu {                               
        proxy_pass http://172.18.0.8:8081;     
    }
...
}
```

![](../images/important/chapter2_custom_nginx_routes.png "Visualisierung wie zwei unterschiedliche URLs beim gleichen Dogu aber in unterschiedlichen Ports enden")

Wenn nun ein Client die URL `https://my-ces-instance/urlx` aufruft, so wird der Request intern auf http://172.18.0.8:8088/neue-url umgeschrieben, das in diesem Beispiel das gleiche Dogu aber mit einem anderen Port darstellt. Die ursprüngliche URL https://my-ces-instance/my-dogu landet wie bisher im gleichen Container-Port. <!-- markdown-link-check-disable-line -->

Zusammenfassend ist dies ist ein mächtiger Mechanismus, um unterschiedliche Routings zu erzeugen.

##### Manuell Einträge modifizieren

In komplexen Szenarien kann das wiederholte Neubauen von Dogu-Container-Images den Entwicklungszyklus verlangsamen. Gleiches gilt, wenn noch nicht klar ist, welche Route genau benötigt wird.

Hier kann eine manuelle Bearbeitung der nginx-Konfiguration helfen, die für die Dogu-Routen zuständig ist.

Die Bearbeitung lässt sich mit Docker- und Container-Bordmitteln erreichen.

```bash
# Datei wie gewünscht bearbeiten
docker exec -it nginx vi /var/nginx/conf.d/app.conf

# nginx-Konfiguration neu einlesen lassen
docker exec -it nginx nginx -s reload -c /etc/nginx/nginx.conf
```

> [!IMPORTANT]
> Diese Änderung hält aber nur solange bis die Datei nicht neu gerendert wird.

Dies passiert i. d. R. wenn:
- neue Dogus installiert werden (neue `location`-Einträge müssen hinzukommen)
- bestehende Dogus deinstalliert werden (Einträge müssen entfernt werden)
- Dogus neustarten (der betreffende `location`-Eintrag wird auf eine HTTP 503-Seite umgeschrieben)

Sollte dies nicht ausreichen, so lässt sich auch eine eigene Datei im Container-Dateisystem ablegen, die nur dann entfernt wird, wenn der Container neu erzeugt wird. Dies sollte eine beliebige `*.conf`-Datei im nginx-Konfigurationsformat im Verzeichnis `/var/nginx/conf.d/app.conf/` sein.
