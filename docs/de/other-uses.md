---
layout: page
title: Andere Anwendungen
lang: de
---

# {{ title }}

Wir haben uns bisher auf Tiles fokussiert. Aber da OpenStreetMap Ihnen – auf einzigartige Weise – Zugriff auf die Roh-Karten-Daten gibt, können Sie beliebige Standort- oder Geo-Anwendung erstellen. Dies sind die gängigsten Anlaufstellen; ein vollständige Liste ist [im OpenStreetMap Wiki](http://wiki.openstreetmap.org/wiki/Frameworks){: target=_blank} verfügbar.

## Gängige Tools

* [Osmosis](http://wiki.openstreetmap.org/wiki/Osmosis){: target=_blank} ist eine universelle Java-Anwendung um OSM-Daten in eine Datenbank zu laden. Die meisten Anwendungen von OSM Daten benutzen Osmosis in irgendeiner Weise.
* [Osmium](http://wiki.openstreetmap.org/wiki/Osmium){: target=_blank} ist ein flexibles Framework, mit schnell steigender Beliebtheit, welches eine stark konfigurierbare Alternative zu Osmosis darstellt.
* [Mapbox Studio](https://www.mapbox.com/mapbox-studio/){: target=_blank} ist eine Sammlung von Werkzeuge um ‘Vector-Tiles’ zu erstellen, die sowohl serverseitig als auch auf Anwender-Seite gerendert werden können.

## Geocoding Dienste

* [Gisgraphy](https://www.gisgraphy.com){: target=_blank} ist ein Open Source Geocoder der eine API / Webdienste anbietet, für Geocodierung und inverse Geocodierung mit Auto-Vervollständigung, Interpolation, Location-Bias, Suche-in-der-Nähe. Alles kann offline oder als gehostete Lösung angewendet werden. Es bietet einige Import-Funktionen für Openstreetmap aber auch Openadresses, Geonames und weitere.
* [Nominatim](https://nominatim.org){: target=_blank} ist die Software hinter OpenStreetMap’s Geocoding Dienst (Ortsname ↔ Lat/Long).
* [OpenCage](https://opencagedata.com/){: target=_blank} bietet eine Geocoding-API, die Nominatim und andere Open-Data Quellen zusammenfasst.
* [OSMNames](https://osmnames.org/){: target=_blank} - Ortsnamen aus OpenStreetMap. Downloadbar. Eingestuft. Mit bbox und Hierarchie. Bereit für Geocodierung.

## Routing-Engines und -Dienste

* [OSRM](http://project-osrm.org/){: target=_blank} ist eine schnelle Routing-Engine, entwickelt für OSM-Daten.
* [Graphhopper](https://github.com/graphhopper/graphhopper/){: target=_blank} ist eine schnelle Java Routing-Engine mit geringem Speicherbedarf.
* [Valhalla](https://valhalla.readthedocs.io/en/latest/){: target=_blank} ist eine C++ Routing-Engine für Straßenfahrzeuge und öffentliche Verkehrsmittel.
* Öffentliche Routing-APIs, die OSM-Daten verwenden, werden von [GraphHopper](https://www.graphhopper.com/products/){: target=_blank}, [MapQuest Open](http://open.mapquestapi.com/directions/){: target=_blank} und [Mapbox](https://www.mapbox.com/directions/){: target=_blank} angeboten.
* Spezielle Routing-APIs bietet [CycleStreets cycle routing](https://www.cyclestreets.net/api/){: target=_blank} (Großbritannien und darüber hinaus)

## Vektor-Karten Bibliotheken (mobil)

* Android Bibliotheken enthalten das [MapLibre Android SDK](https://maplibre.org/projects/maplibre-native/){: target=_blank}, [Mapbox Android SDK](https://www.mapbox.com/android-sdk/){: target=_blank}, [mapsforge](http://mapsforge.org/){: target=_blank}, [Nutiteq Maps SDK](https://developer.nutiteq.com/){: target=_blank}, [Skobbler Android SDK](http://developer.skobbler.com/){: target=_blank} und [Tangram ES](https://github.com/tangrams/tangram-es/){: target=_blank}.

* iOS Bibliotheken enthalten das [MapLibre iOS SDK](https://maplibre.org/projects/maplibre-native/){: target=_blank}, [Mapbox iOS SDK](https://www.mapbox.com/ios-sdk/){: target=_blank}, [Nutiteq Maps SDK](https://developer.nutiteq.com/){: target=_blank}, [Skobbler iOS SDK](http://developer.skobbler.com/){: target=_blank} und [Tangram ES](https://github.com/tangrams/tangram-es/){: target=_blank}.

## Vektor-Karten Bibliotheken (Web)

* [MapLibre GL JS](https://maplibre.org/projects/maplibre-gl-js/){: target=_blank}, [Mapbox GL JS](https://www.mapbox.com/mapbox-gl-js/){: target=_blank} und [Tangram](http://tangrams.github.io/tangram/){: target=_blank} rendern Vector-Tiles basierend auf OSM-Daten unter Verwendung von WebGL für bessere Leistungsfähigkeit.
