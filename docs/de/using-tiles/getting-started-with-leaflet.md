---
layout: docs
title: Erste Schritte mit Leaflet
---

# {{ title }}

## Einleitung

[Leaflet](http://leafletjs.com/){: target=blank} ist eine leichtgewichtige JavaScript-Library zur Einbindung von Karten. Es verwendet eine freizügige BSD Open-Source Lizenz - kann also ohne Bedenken legal in jede Seite integriert werden. Der Quellcode ist auf [GitHub](http://github.com/Leaflet/Leaflet){: target=blank} verfügbar.

Wir beschränken uns hier auf ein kleines, eigenständiges Beispiel und verweisen für komplexere Anwendungen auf die offiziellen [Tutorials](http://leafletjs.com/examples.html){: target=blank} und die [Dokumentation](http://leafletjs.com/reference.html){: target=blank}.

## Erste Schritte

Kopieren Sie den folgenden Inhalt in eine Datei [leaflet.html](leaflet.html){: target=_blank} und öffnen Sie sie in Ihrem Browser:

``` html title="leaflet.html"
--8<-- "docs/en/using-tiles/leaflet.html"
```

Für eine detaillierte Erklärung des Codes, besuchen Sie die offizielle Webseite. [^1]

[^1]: Quick Start&nbsp;– <https://leafletjs.com/examples/quick-start/>{: target=blank}

## Weitere Links

Sie möchten …

* einen anderen Hintergrund verwenden? → Leaflet unterstützt standardmäßig [TMS](https://de.wikipedia.org/wiki/Tile_Map_Service){: target=blank} und [WMS](https://de.wikipedia.org/wiki/Web_Map_Service){: target=blank}. Siehe [hier](http://leafletjs.com/reference.html#tilelayer){: target=blank} welche Optionen in Leaflet unterstütz werden.
* alle Standorte Ihres Unternehmens hinzufügen? → Stellen Sie sie als [GeoJSON](http://geojson.org/){: target=blank} bereit und [integrieren](http://leafletjs.com/examples/geojson.html){: target=blank} Sie sie in die Karte.
* eine andere Kartenprojektion verwenden? → Verwenden Sie das [Proj4Leaflet](https://github.com/kartena/Proj4Leaflet){: target=blank} Plugin.
