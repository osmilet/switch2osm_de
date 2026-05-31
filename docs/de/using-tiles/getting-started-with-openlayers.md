---
layout: docs
title: Erste Schritte mit OpenLayers
lang: de
---

# {{ title }}

## Einleitung

[OpenLayers](http://openlayers.org/){: target=blank} ist eine vollständige JavaScript-Library zur Einbindung von Karten. Es verwendet eine freizügige BSD Open-Source Lizenz - kann also ohne Bedenken legal in jede Seite integriert werden. Der Quellcode ist auf [GitHub](https://github.com/openlayers/ol3/){: target=blank} verfügbar.

Wir beschränken uns hier auf ein kleines, eigenständiges Beispiel und verweisen für komplexere Anwendungen auf die offiziellen [Tutorials](http://openlayers.org/en/latest/examples/) und die [API](http://openlayers.org/en/latest/apidoc/).

## Erste Schritte

Ein Beispiel zur Bereitstellung einer Karte mit Hilfe der OpenLayers-Library wird im [Quick Start](https://openlayers.org/doc/quickstart.html){: target=blank}-Abschnitt der OpenLayers-Dokumentation beschrieben. Die Installation des [node.js](https://nodejs.org/){: target=blank} Frameworks und die Installation der Library erfordert ein wenig Mühe, daher werden wir dies hier nicht weiter ausführen.

## Weitere Links

Sie möchten …

* einen anderen Hintergrund verwenden? → Openlayers unterstützt standardmäßig [TMS](https://de.wikipedia.org/wiki/Tile_Map_Service){: target=blank} und [WMS](https://de.wikipedia.org/wiki/Web_Map_Service){: target=blank}. Siehe [Openlayers offizielle Beispiele](http://openlayers.org/en/latest/examples/){: target=blank} und [die API](http://openlayers.org/en/latest/apidoc/){: target=blank} um zu erfahren, welche Optionen unterstützt werden.
* alle Standorte Ihres Unternehmens hinzufügen? → Stellen Sie sie als [GeoJSON](http://geojson.org/){: target=blank} bereit und [integrieren](http://openlayers.org/en/latest/examples/select-features.html){: target=blank} Sie sie in die Karte.
* eine andere Kartenprojektion verwenden? → OpenLayers unterstützt alle Proj4-Projektionen, sofern Sie die [proj4js](http://proj4js.org/){: target=blank} JavaScript-Library einbinden. Darüber hinaus wird [Client-seitige Raster Re-Projektion](http://openlayers.org/en/latest/examples/reprojection-by-code.html){: target=blank} unterstützt, wodurch Sie OpenStreetMap-Tiles in Ihrer lokalen Projektion verwenden können.
