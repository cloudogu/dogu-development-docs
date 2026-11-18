# Dogu-V3-Dokumentation

Dieser Bereich beschreibt die Entwicklung und den Betrieb von Dogu-V3-Anwendungen.

## Lernpfad

Die Dokumentation führt Entwickler:innen von einem minimalen lauffähigen Dogu zu detaillierten, aufgabenorientierten Informationen:

1. die lokale Entwicklungsumgebung vorbereiten,
2. mit einem Quickstart ein minimales Dogu V3 bereitstellen,
3. Artefakte und Lebenszyklus verstehen,
4. die relevante Multinode-Umgebung verstehen,
5. CES-Integrationen mit gezielten Anleitungen ergänzen und
6. bei Bedarf Informationen zu Release, Migration, Upgrades oder Fehlerbehebung nachschlagen.

## Konzepte

- [Dogu-V3-Artefakte verstehen](concepts/artifacts_de.md)
- [Die Multinode-Laufzeitumgebung verstehen](concepts/multinode-environment_de.md)

## Erste Schritte

- [Voraussetzungen für die Dogu-Entwicklung](getting-started/prerequisites_de.md)
- [Vom Helm-Chart zum Dogu V3](getting-started/quickstart_de.md)
- [XWiki in das CES integrieren](getting-started/advanced-guide_de.md)

## Anleitungen

- [Ein Dogu V3 releasen](guides/release_de.md)
- [Ein V2-Dogu als migrierbares V3-Chart bereitstellen](guides/v2-v3-migration_de.md)
- [Backup & Restore eines Dogus testen](guides/backup-restore_de.md)
- Upgrades eines Dogu V3 (geplant)

Neue Anleitungen folgen der [Vorlage für Anleitungen](guides/_template_de.md).

## Best Practices

- [Best Practices für Dogu-Helm-Charts](best-practices/helm-chart_de.md) (wird erweitert)

## Fehlerbehebung

- Fehlerbehebung für Dogu V3 (geplant)

## Referenz

- [Kompendium der Dogu-V3-Artefakte](reference/compendium_de.md)

## Inhaltliche Abgrenzung

- **Erste Schritte** enthalten ausführbare, durchgängige Anleitungen. Der Quickstart behandelt nur den kleinsten lauffähigen Happy Path.
- **Konzepte** erklären das mentale Modell und Zusammenhänge, ohne technische Referenzen zu duplizieren.
- **Anleitungen** erklären, wie eine konkrete Aufgabe der Dogu-Entwicklung erledigt wird.
- **Referenz** bietet kompakte Informationen zum Nachschlagen und verlinkt für vollständige API-Verträge auf die verantwortlichen Repositories.
- **Best Practices** enthalten nur Cloudogu- und CES-spezifische Hinweise.
- **Fehlerbehebung** beginnt bei beobachtbaren Symptomen und verlinkt bei Bedarf auf komponentenspezifische Diagnosen.

Was ein anderes Repository bereits dokumentiert, wird verlinkt statt wiederholt. Das gilt für Werkzeuge wie [`cldk`](https://github.com/cloudogu/cldk/blob/develop/README.md), für die Dokumentation der Cloudogu-Operatoren und CRDs sowie für allgemeine Helm-, Kubernetes-, OCI- und Container-Dokumentation. Links in Cloudogu-Repositories zeigen auf den Branch `develop`.

## Abgrenzung zu Dogu V2

- Alle bestehenden Seiten in `docs/core`, `docs/important` und `docs/additional` beschreiben Dogu V2, sofern eine Seite nicht ausdrücklich etwas anderes angibt.
- Natives Dogu-V3-Verhalten wird ausschließlich unterhalb von `docs/v3` dokumentiert.
