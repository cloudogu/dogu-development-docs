# Relevante Funktionalitäten
Dieses Kapitel beschreibt die Features und mögliche Implementierungsideen/-Lösungen derer Funktionalitäten, die überhaupt ein echtes Dogu ausmachen und wiederkehrende Probleme lösen. 

Die folgenden Abschnitte beschäftigen sich daher mit wiederkehrenden Funktionen, wie ein Dogu sich in die Landschaft des Cloudogu EcoSystem einbetten kann.

## Authentifizierung
Lorem ipsum Einführungstext

### CAS-Protokoll


### OAuth-Protokoll


### OIDC-Protokoll


## Auf die Registry zugreifen


## Aufbau und Best Practices von `startup.sh` 


## Service Accounts
Lorem ipsum Einführungstext

### Service Accounts produzieren 


### Service Accounts konsumieren

## Dogu-Upgrades


## Typische Dogu-Features

Dieses Kapitel beschreibt Funktionen, die Dogus tiefer in das Cloudogu EcoSystem integrieren und sie einheitlich administrierbar machen.

### Memory-/Swap-Limit

Mit Memory- und Swap-Limits kann man den Speicherverbrauch (Arbeitsspeicher und Swap) von Dogus einschränken.

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

Hiermit lassen sich die etcd-Einträge `container_config/memory_limit` und `container_config/swap_limit´ in der jeweiligen Dogu-Konfiguration erstellen.

Die konfigurierbaren Werte für die Schlüssel sind jeweils eine Zeichenkette der Form `<Zahlenwert><Einheit>` und beschreiben die maximal vom Dogu nutzbare Menge an Speicher.
Zu beachten ist hier, dass zwischen dem numerischen Wert und der Einheit kein Leerzeichen stehen darf.
Verfügbare Einheiten sind `b`, `k`, `m` und `g` (für byte, kibibyte, mebibyte und gibibyte).

Das Setzen der Werte kann über folgende Wege erfolgen:
- `doguctl config container_config/memory_limit 1g`
- `cesapp edit-config <doguname>` (nur von Host aus)
- `etcdctl set /config/<doguname>/container_config/memory_limit "1g"` (nur von Host aus)

Um die Limitierungen zu übernehmen, muss das Dogu neu erstellt (`cesapp recreate <doguname>`) und anschließend neu gestartet (`cesapp start <doguname>`) werden.

Falls ein Dogu seine Speicherlimitierung überschreitet, wird der größte Prozess im Container beendet.
Dies ist normalerweise der Hauptprozess des Dogus und führt dazu, dass der Container neu gestartet wird.

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

Die damit konfigurierbaren Werte müssen in den Start-Skripten des Dogus dem entsprechenden Java-Prozess als Parameter mitgegeben werden. Eine Referenzimplementierung findet sich im Nexus-Dogu: https://github.com/cloudogu/nexus/blob/77bdcfdbe0787c85d2d9b168dc38ff04b225706d/resources/util.sh#L52



### Backup & Restore-Fähigkeit


### Änderbarkeit der Admin-Gruppe


### Änderbarkeit der FQDN


### Loglevel mappen und ändern

