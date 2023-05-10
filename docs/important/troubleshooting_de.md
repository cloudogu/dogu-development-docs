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

Besteht eine ungefähre Vermutung zur Position der fehlerhaften Stelle, können zusätzliche `echo` - Ausgaben helfen das Problem zu analysieren.

Falls weiterhin die Fehlerquelle unklar bleibt, ist es notwendig die Anweisungen innerhalb des Containers einzeln auszuführen:
- Shell in den Container: `docker exec -it <doguname> bash`
- Anschließend kann der Container debuggt werden:
  - Prüfung des Filesystems
  - Iterative Ausführung der Befehle in den Skripten, um den Fehler zu lokalisieren
  - Defekt mittels Bordmitteln im Container reparieren

Bei einer Restart-Loop haben sie folgende Möglichkeiten den Container in einen stabilen Zustand zu bringen:
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

Es ratsam die Größe der Volumes regelmäßig zu überprüfen.
Diese werden unter dem Pfad `/var/lib/ces/<dogu>/volumes/` gemounted.

Ein Dogu sollte einen wachsenden Datenbestand ausschließlich in Docker Volumes speichern, die in der [`dogu.json`](https://github.com/cloudogu/dogu-development-docs/blob/main/docs/core/compendium_en.md#volumes) beschrieben werden. 

Volumes haben neben einer besseren Geschwindigkeit weitere Vorteile: Sie gehen nicht verloren, wenn der Container beendet wird. Außerdem lässt sich die Festplattenauslastung besser überwachen.

## Wo finde ich FQDN, Zertifikat und weitere allgemeine Konfigurationen?

siehe: [Registryeinträge](relevant_functionalities_de.md#weitere-registryeinträge)

## Wie werden Registryeinträge gelesen/geschrieben?

Mittels [doguctl](relevant_functionalities_de.md#die-nutzung-von-doguctl).
Es ist in unserem [Base-Image](https://github.com/cloudogu/base) enthalten.

## Das Dogu ist nicht über Nginx erreichbar

- Wurde diese Umgebungsvariable im Dockerfile gesetzt?
  - `ENV SERVICE_TAGS=webapp`
- Konfiguration im Nginx-Container checken
  - `docker exec -it nginx bash`
  - `cat /etc/nginx/conf.d/app.conf`

## Dogu erscheint nicht im Warp Menü

- Für einen Eintrag im Warp-Menü muss das Tag `warp` und eine Kategorie in der dogu.json definiert werden.
- Das Dogu muss gültiges HTML mit einem abschließendem `<\html>`-Tag ausliefern.
- Das Dogu wurde wegen einer fehlerhaften Installation nicht registriert.

## Dogu-Volumes werden nicht von Backup & Restore beachtet

Damit Volume-Daten eines Dogus von dem Backup & Restore Mechanismus beachtet werden muss das entsprechende Volume mit dem Feld `NeedsBackup` erweitert werden:

```json
{
  "Name": "db",
  "Path": "/var/lib/openldap",
  "NeedsBackup": true
}
```





