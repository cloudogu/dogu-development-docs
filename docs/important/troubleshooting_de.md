# Troubleshooting

## Logging-Ausgaben fehlen oder sind zu präzise oder zu unpräzise

- Es wird ein falscher Pfad zum Auslesen des Log-Levels verwendet
  - Der korrekte Pfad lautet: `config/<doguname>/logging/root`
- Das Log-Level ist ungültig
  - Valide Werte: `ERROR`, `WARN`, `INFO`, `DEBUG`
- Der Wert wird beim Starten des Dogus nicht sinnvoll interpretiert. Ein korrektes Beispiel wäre:
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

## Wie vermeide ich Fehler aufgrund von ungültigen Konfigurationseinträgen?

- Gültige ETCD-Pfade sind zu beachten
- `doguctl` kann Konfigurationswerte validieren siehe [Validierung](relevant_functionalities_de.md#validierung-und-default-werte)
- Übersicht aller Keys: `etcdctl ls -r config/<doguname>`
- Aus dem Container ist es möglich verschlüsselte Werte zu überprüfen:
  - `docker exec -it redmine bash`
  - `doguctl config -e sa-postgresql/username`

## Das Dogu startet nicht und/oder befindet sich in einer Restart-Loop

Besteht eine ungefähre Vermutung zur Position der fehlerhaften Stelle, reicht es oftmals das Skript mit `echo` - Ausgaben zu erweitern,
das Dogu neu zu starten und anschließend das Log-File zu prüfen.

Falls nicht, ist es notwendig den Container zu debuggen:
- Shell in den Container: `docker exec -it <doguname> bash`
- Bei einer Restart-Loop überschreiben sie am besten das Startskript des Dockerfiles, sodass der Container schläft:
  - `ENTRYPOINT ["tail", "-f", "/dev/null"]` anstatt `CMD ["/resources/startup.sh"]`
  - Alternativ kann an beliebiger Stelle aus ein `sleep <time>` verwendet werden
- Anschließend kann der Container debuggt werden:
  - Prüfung des Filesystems
  - Iterative Ausführung der Befehle in den Skripten um den Fehler zu lokalisieren


## Volumes

Es ratsam die Größe der Volumes regelmäßig zu überprüfen.
Diese werden unter dem Pfad `/var/lib/ces/` gemounted.

Das Dogu sollte einen wachsenden Datenbestand außerdem auch nur in Volumes speichern, weil sonst
die Festplatte des Systems voll laufen könnte.

## Wo finde ich FQDN, Zertifikat und weitere allgemeine Konfigurationen?

siehe: [Registryeinträge](relevant_functionalities_de.md#weitere-registryeinträge)

## Wie werden Registryeinträge gelesen/geschrieben?

Mittels [doguctl](https://github.com/cloudogu/doguctl).
Es ist in unserem [Base-Image](https://github.com/cloudogu/base) enthalten.

## Das Dogu ist nicht über Nginx erreichbar

- Umgebungsvariable im Dockerfile prüfen
  - `ENV SERVICE_TAGS=webapp`
- Konfiguration im Nginx-Container checken
  - `docker exec -it nginx bash`
  - `cat /etc/nginx/conf.d/app.conf`

## Dogu erscheint nicht im Warp Menü

- Für einen Eintrag im Warp-Menü muss das Tag `warp` und eine Kategorie in der dogu.json definiert werden.

## Dogu-Volumes werden nicht von Backup & Restore beachtet

Damit Volume-Daten eines Dogus von dem Backup & Restore Mechanismus beachtet werden muss das entsprechende Volume mit dem Feld `NeedsBackup` erweitert werden:

```json
{
  "Name": "db",
  "Path": "/var/lib/openldap",
  "NeedsBackup": true
}
```





