---
layout: docs
title: Eigene Tiles bereitstellen
lang: de
---

# {{ title }}

Tiles von einem Drittanbieter sind der einfachste Weg den Wechsel zu OpenStreetMap umzusetzen und bieten Kostentransparenz. Wenn Sie jedoch gerne die volle Kontrolle über Ihr Schicksal übernehmen wollen, können Sie Ihre eigenen Tiles rendern und ausliefern. Dieser Abschnitt erklärt wie.

![raw osm data](/assets/img/raw-osm-data.webp){ data-title="OSM Roh-Daten" data-description="Raw OpenStreetMap data with the default map paint style in JOSM, retrieved via OSM API for editing. https://www.openstreetmap.org/#map=16/18.0253/-63.0485" } | ![tile server](/assets/img/vector_tiles_pyramid_structure_window.webp){ data-title="Karten-Tiles-Pyramide" data-description="Jede Zoom-Ebene der Karte ist in kleine Stücke, sogenannte Tiles, unterteilt. Die Größe eines Tiles liegt üblicherweise bei 256×256 Pixel." } | ![map usage](/assets/img/map-usage.webp){ data-title="Tiles werden von Ihrer Webseite angeboten" data-description="Die von Ihrem Tile-Server vorbereiteten Tiles werden im Web-Browser des Clients, oder einer anderen Anwendung, angezeigt." }
:--:|:--:|:--:
:simple-openstreetmap: :material-database-import: OpenStreetMap Roh-Daten | :material-server: :material-checkerboard-plus: Ihr eigenen Tile-Server | :fontawesome-solid-users: :octicons-browser-16: Nutzer, die Ihre Webseite besuchen

## Passt es zu Ihnen?

Erzeugen und Anbieten von Tiles verursacht erhebliche Hardware-Anforderungen, insbesondere wenn globale Abdeckung und regelmäßige Aktualisierungen benötigen werden.

Wenn Sie Ihren eigenen Tile-Server einrichten, empfehlen wir Ihnen [Ubuntu Linux](https://ubuntu.com/){: target=_blank} oder [Debian](https://www.debian.org/releases/){: target=_blank} zu nutzen.

## Die Optionen

1. Lokal installieren auf [Debian 13](manually-building-a-tile-server-debian-13.md), [Debian 12](manually-building-a-tile-server-debian-12.md), [Ubuntu 24.04](manually-building-a-tile-server-ubuntu-24-04-lts.md), oder [Debian 11](manually-building-a-tile-server-debian-11.md).

2. [Docker](using-a-docker-container.md) verwenden.

## Systemanforderungen

Ihre eigenen Karten anzubieten stellt eine ziemlich anspruchsvolle Aufgabe dar. Abhängig von der Größe des zu verteilenden Gebiets und des zu erwartenden Datenverkehrs, unterscheiden sich die System-Anforderungen. Im Allgemeinen liegen die Anforderungen im Bereich von 10-20GB Speicherplatz, 4GB Arbeitsspeicher und einem modernen Dual-Core-Prozessor für ein Gebiet der Größe einer Stadt bis zu 1TB schnellem Speicherplatz, 24GB Arbeitsspeicher und einem Quad-Core-Prozessor für den gesamten Planeten.

Wir würden empfehlen, dass Sie mit Extrakten von OpenStreetMap-Daten starten – zum Beispiel eine Stadt, ein Landkreis oder eine kleine Stadt – anstatt eine Woche damit zu verbringen die ganze Welt (planet.osm) zu importieren und dann auf Grund eines Konfigurations-Fehlers neu beginnen zu müssen! Sie können Extrakte herunterladen von:

* [Geofabrik](https://download.geofabrik.de/){: target=_blank} (Länder und Regionen)
* [slice.openstreetmap.us](https://slice.openstreetmap.us){: target=_blank} (minütlich aktualisierte Städte und kleine Länder)
* [download.openstreetmap.fr](https://download.openstreetmap.fr/){: target=_blank}

## Die Toolchain

Wir benutzen eine Reihe an Werkzeugen zum Generieren und Anbieten von Karten-Tiles.

**Apache** liefert den Front-End-Server, der Anfragen von Ihrem Webbrowser verarbeitet und die Anfragen an mod_tile weiterleitet. Der Apache Web-Server kann ebenfalls verwendet werden um statische Web-Inhalte wie HTML, JavaScript oder CSS für Ihre Web auszuliefern.

Sobald Apache die Anfrage vom Web-Nutzer verarbeitet, übergibt es die Anfrage an mod_tile damit dieses sich darum kümmert. Mod_tile überprüft, ob der Tile bereits erzeugt wurde und zur Verwendung bereit ist oder ob er aktualisiert werden muss, weil er sich noch nicht im Cache befindet. Wenn er bereits verfügbar ist und nicht erst noch gerendert werden muss, dann wird er umgehend an den Client zurück gesendet. Wenn er zunächst noch gerendert werden muss, dann wird er zur “Render Request”-Warteschlange hinzugefügt. Und wenn er das Ende der Warteschlange erreicht, wird ein Tile-Renderer ihn rendern und den Tile an den Client zurück senden.

Wir verwenden ein Werkzeug namens **Mapnik** um Tiles zu rendern. Es entnimmt so schnell wie möglich Anfragen aus der Arbeits-Warteschlange, extrahiert Daten aus verschiedenen Datenquellen gemäß den Stil-Informationen und rendert den Tile. Dieser wird an den Client zurückgegeben und dann mit dem nächsten in der Warteschlange begonnen.

Für den Rendering-Prozess werden OpenStreetMap-Daten in einer **PostgreSQL** Datenbank gespeichert, die durch ein Werkzeug namens **osm2pgsql** erstellt wird. Diese zwei Teile arbeiten zusammen um einen effizienten Zugriff auf die geographischen OpenStreetMap-Data zu erlauben. Es ist möglich die Daten in der PostgreSQL-Datenbank aktuell zu halten, indem ein Stream mit Diff-Files verwendet wird, der alle 60 Sekunden auf dem OpenStreetMap-Hauptserver generiert wird.
