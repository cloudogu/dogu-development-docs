# Best Practices für Dogu-Helm-Charts

Dieses Dokument beschreibt die Best-Practices, die es bei der Erstellung von Dogu-Helm-Charts zu beachten gilt.

## Vorsicht beim Umbenennen von Kubernetes-Ressourcen

Eine Änderung von `metadata.name` erzeugt ein anderes Kubernetes-Objekt. Die Auswirkungen hängen von Ressourcenart und Controller ab: Referenzen können ungültig werden, ein Controller kann neue Credentials ausstellen und zustandsbehaftete Ressourcen können getrennt oder verwaist werden. Aus einer Umbenennung folgt nicht automatisch, dass Kubernetes die zugrunde liegenden Daten löscht.

Prüfen Sie vor einer Namensänderung diese Beziehungen:

| Beziehung | Typisches Risiko |
| --- | --- |
| PVC- und Claim-Referenzen | Ein Workload startet nicht, wenn der referenzierte Claim fehlt. Ein neuer Claim kann an einen anderen Speicher gebunden werden; ob das bisherige Volume und der zugrunde liegende Speicher nach dem Löschen des alten Claims erhalten bleiben, hängt von der Reclaim Policy des PersistentVolume ab. |
| Service, Clients und Selektoren | Clients und `Exposition`-Ressourcen können weiterhin den alten Service referenzieren; geänderte Selektoren können außerdem dazu führen, dass ein Service keine Endpoints mehr besitzt. |
| Exposition-Ziel | Externer Zugriff kann auf einen nicht mehr vorhandenen Service oder Port zeigen. |
| ServiceAccountRequest und verwaltetes Secret | Ein neuer Request kann Credentials neu erzeugen, während die Anwendung noch das alte Secret erwartet. |
| Andere Integrationsressourcen | Authentifizierungs- oder Menüintegration kann dupliziert, ersetzt oder getrennt werden. |
| Backup und Aufbewahrung | Prüfen Sie Selektionslabels sowie Restore- und Aufbewahrungsverhalten der betroffenen Ressource. |

Behandeln Sie die Umbenennung zustandsbehafteter oder extern referenzierter Ressourcen als **Migration**: Aktualisieren Sie alle Referenzen, prüfen Sie Backup- und Aufbewahrungsverhalten und entfernen Sie das alte Objekt erst, nachdem der neue Pfad validiert wurde.
