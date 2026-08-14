# Dogu-V3-Dokumentation

Dieser Bereich wird Dogu-Entwickler:innen durch die Entwicklung und den Betrieb von Dogu-V3-Anwendungen führen.

## Geplanter Lernpfad

Die Dokumentation wird Entwickler:innen von einem minimalen lauffähigen Dogu zu detaillierten, aufgabenorientierten Informationen führen:

1. die lokale Entwicklungsumgebung vorbereiten,
2. mit einem Quickstart ein minimales Dogu V3 bereitstellen,
3. Artefakte und Lebenszyklus verstehen,
4. die relevante Multinode-Umgebung verstehen,
5. CES-Integrationen mit gezielten Anleitungen ergänzen und
6. bei Bedarf Informationen zu Release, Migration oder Fehlerbehebung nachschlagen.

## Verfügbare Konzepte

- [Dogu-V3-Artefakte verstehen](concepts/artifacts_de.md)
- [Die Multinode-Laufzeitumgebung verstehen](concepts/multinode-environment_de.md)

## Verfügbare Referenz

- [Kompendium der Dogu-V3-Artefakte](reference/compendium_de.md)

## Inhaltliche Abgrenzung

- **Erste Schritte** enthalten ausführbare, durchgängige Anleitungen. Der Quickstart behandelt nur den kleinsten lauffähigen Happy Path.
- **Konzepte** erklären das mentale Modell und Zusammenhänge, ohne technische Referenzen zu duplizieren.
- **Anleitungen** erklären, wie eine konkrete Aufgabe der Dogu-Entwicklung erledigt wird.
- **Referenz** bietet kompakte Informationen zum Nachschlagen und verlinkt für vollständige API-Verträge auf die verantwortlichen Repositories.
- **Best Practices** enthalten nur Cloudogu- und CES-spezifische Hinweise. Allgemeine Helm-, Kubernetes-, OCI- und Container-Dokumentation wird verlinkt statt neu geschrieben.
- **Fehlerbehebung** beginnt bei beobachtbaren Symptomen und verlinkt bei Bedarf auf komponentenspezifische Diagnosen.

## Abgrenzung zu Dogu V2

- Alle bestehenden Seiten in `docs/core`, `docs/important` und `docs/additional` beschreiben Dogu V2, sofern eine Seite nicht ausdrücklich etwas anderes angibt.
- Natives Dogu-V3-Verhalten wird ausschließlich unterhalb von `docs/v3` dokumentiert.
