---
layout: docs
title: Tiles verwenden
lang: de
---

# {{ title }}

Sie können eine Website in weniger als einer Stunde auf OpenStreetMap umstellen. Wählen Sie eine JavaScript API und Tile-Anbieter, und Sie sind startklar. Dann, wenn sich Ihre Anforderungen erhöhen, können Sie über benutzerdefinierte Tiles nachdenken. Entweder von einem spezialisierten Anbieter oder von Ihnen selbst erzeugt.

## Eine API/Library auswählen

Anders als kommerzielle Online-Karten-Anbieter, bietet OpenStreetMap keine “offizielle” JavaScript-Library die Sie nutzen müssen. Stattdessen können Sie jede Library verwenden, die Ihren Anforderungen entspricht. Die beliebteste ist Leaflet, eine Open-Source Library. OpenLayers 3, eine weitere sehr bekannte Library, kann ebenfalls gut geeignet sein.

[Erste Schritte mit Leaflet – eine leichtgewichtige Web-Karten-Library](/using-tiles/getting-started-with-leaflet.md)

[Erste Schritte mit Openlayers – eine Funktions-vollständige Library für Web-Karten](/using-tiles/getting-started-with-openlayers.md)

[Erste Schritte mit Maplibre](/using-tiles/getting-started-with-maplibre.md)

## Einen Tile-Anbieter auswählen

Von streng begrenzten Testzwecken abgesehen, sollten Sie die von OpenStreetMap.org selbst bereitgestellten Tiles nicht verwenden. OpenStreetMap ist eine Freiwilligen-betriebene, gemeinnützige Einrichtung und kann Tiles für eine umfangreiche kommerzielle Nutzung nicht bereitstellen. Stattdessen sollten Sie einen Drittanbieter verwenden, der Tiles aus OSM-Daten erstellt, oder Ihre eigenen erstellen.

### Kostenlose Anbieter

Sie können eine Liste über die Vorschau des Projekts [Leaflet-provider](http://leaflet-extras.github.io/leaflet-providers/preview/) erhalten. Einige davon sind jedoch nicht frei verfügbar (benötigen einen API-Schlüssel).

### Kostenpflichtige Anbieter

Siehe [Liste](/providers.md). Oder lesen Sie weiter, um herauszufinden wie Sie ihre eigenen Tiles erzeugen und anbieten.
