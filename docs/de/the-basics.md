---
layout: page
title: Die Grundlagen
lang: de
---

# {{ title }}

## Die Herausforderung

Ihr aktueller Kartenanbieter gibt Ihnen zwei Dinge:

* Eine Anzahl von `Tiles` (quadratische Karten-Grafiken), die zusammengefügt eine Karte bilden
* Eine JavaScript API, oder gleichwertige Bibliothek für Mobile Apps, zur Anzeige der Tiles

Um zu OpenStreetMap zu wechseln, müssen Sie beides ersetzen.

## Die Tiles

![Tiles](/assets/img/tiles.webp){ align=left .off-glb }
Die Karten-Tiles, Grafiken in (üblicherweise) 256 x 256 Pixel Größe, werden aus einer Karten-Datenbank erzeugt (“rendered”).

Wenn Sie momentan Google Maps nutzen, verwenden Sie Google’s Karten-Tiles, die bei google.com gehosted werden. Da die OpenStreetMap Stiftung eine gemeinnützige Organisation mit begrenzten Ressourcen ist, können Sie nicht einfach die Tiles von openstreetmap.org als Ersatz übernehmen (siehe [Tile Usage Policy](https://operations.osmfoundation.org/policies/tiles/){: target=blank}). Stattdessen können Sie:

* Ihre eignen Tiles generieren, indem Sie die freie OSM Karten-Datenbank herunterladen und rendern;

* Oder einen Drittanbieter verwenden (manche davon gebührenpflichtig, andere kostenlos)

Die OSM-Karten-Datenbank wird planet.osm genannt. Die vollständige Datenbank und regelmäßige Update-Dateien sind beide über [planet.openstreetmap.org](http://planet.openstreetmap.org/){: target=blank} verfügbar.

Das Rendern Ihrer eigenen Tiles gibt Ihnen volle Kontrolle über deren Erscheinungsbild. Sie können die Karten individuell anpassen, um sie aussehen zu lassen wie Sie möchten. Alternativ haben Drittanbieter OSM-Kompetenz und könnten fertig vorbereitete Karten-Stile bereithalten, die Sie nutzen können.

### Raster-Tiles oder Vector-Tiles

Raster-Tiles und Vector-Tiles sind zwei unterschiedliche Ansätze um Karten-Daten darzustellen und anzubieten. Jeder davon hat seine eigenen Vorteile und Anwendungszwecke. Lassen Sie uns die Unterschiede zwischen Raster-Tiles und Vector-Tiles erkunden um ihre Stärken und Einschränkungen zu verstehen.

**Raster-Tiles**

Raster-Tiles sind im Wesentlichen Grafiken oder Bilder von Karten-Daten. Sie sind für verschiedene Zoom-Stufen vorgerendert und als eigenständige Grafikdateien gespeichert. Hier sind einige Hauptmerkmale von Raster-Tiles:

* Raster-Tiles stellen Karten-Daten als ein Gitter aus Pixeln dar. Jede Tile ist eine statische Grafik, die einen Abschnitt der Karte bei einer bestimmten Zoom-Stufe abbildet.
* Raster-Tiles haben ein festgelegtes Erscheinungsbild, da sie mit vordefinierten Stilen erzeugt werden. Um to die visuelle Darstellung der Karte zu ändern, müssen neue Tiles gerendert werden, was rechenintensiv sein kann.
* Raster-Tiles können höhere Dateigrößen verglichen zu Vector-Tiles haben, weil sie Details auf Pixel-Ebene für jede Tile speichern, was zu höheren Speicher-Anforderungen und langsameren Downloadraten führt.
* Raster-Tiles eignen sich besonders gut zur Darstellung von komplexen kartographischen Stilen, wie topographischen Karten oder Satellitenaufnahmen, bei denen kleine Details wichtig sind.
* Raster-Tiles bieten begrenzte interaktive Möglichkeiten, hauptsächlich beschränkt auf einfaches Zoomen und Verschieben. Interaktion mit individuellen Kartenmerkmalen oder dynamische Modifikation der Kartendarstellung ist anspruchsvoll.

**Vector-Tiles**

Vector-Tiles, auf der anderen Seite, stellen Karten-Daten als eine Sammlung geometrischer Eigenschaften dar. Beispielsweise Punkte, Linien und Polygone. Dies sind die Besonderheiten von Vector-Tiles:

* Vector-Tiles speichern Karten-Daten als individuelle Geometrien und Attribute. Diese Geometrien können in Echtzeit skaliert, rotiert und umgestaltet werden, wodurch sie mehr Flexibilität und Anpassungsmöglichkeiten bieten.
* Vector-Tiles erlauben dynamische Gestaltung und Modifikation von Karten Eigenschaften. Stile können on-the-fly gewechselt werden, einschließlich Farben, Linienbreiten, Platzierung von Beschriftungen und andere visuelle Eigenschaften.
* Vector-Tiles sind üblicherweise kleiner, verglichen zu Raster-Tiles. Da sie nur Geometrien, Daten und Attribute speichern, benötigen sie weniger Speicherplatz und führen so zu kürzeren Übertragungszeiten.
* Vector-Tiles benötigen weniger Bandbreite zur Übertragung, weil nur die notwendigen Kartendaten zum Empfänger gesendet werden. Dies ist insbesondere für mobile Anwendungen oder Regionen mit eingeschränkter Internetversorgung vorteilhaft.
* Vector-Tiles ermöglichen umfangreiche Interaktivität und Echtzeit-Rendering. Anwender können mit individuellen Karten Elementen interagieren, dynamische Abfragen ausführen und Eigenschafts-basierte benutzerdefinierte Gestaltungsstile anwenden. Dadurch ermöglicht sich ein interaktiveres und personalisiertes Kartenerlebnis.

### Anwendungsfälle

Die Entscheidung zwischen Raster-Tiles und Vector-Tiles hängt von spezifischem Anwendungsfall und Anforderungen ab. Hier sind einige Szenarien für die sich eine von beiden Varianten besonders eignet:

**Raster-Tiles**

* Ästhetisch detaillierte Karten, wie topographische Karten oder Satellitenaufnahmen.
* Statische Karten, die keine Echtzeit-Interaktivität oder oder häufige Aktualisierungen benötigen.
* Fälle in denen die Karten-Daten relativ beständig sind und keine häufigen Modifikationen oder Gestaltungsänderungen benötigen.

**Vector-Tiles**

* Dynamische Karten die Echtzeit-Anpassung und Interaktivität erfordern, wie etwa anwendergesteuerte Gestaltung oder Filterung.
* Mobile Anwendungen oder Bereiche mit eingeschränkter Bandbreite oder Speicherkapazität.
* Karten mit sich häufig ändernden Daten, bei denen Updates in Echtzeit wiedergegeben müssen.

Sowohl Raster-Tiles als auch Vector-Tiles haben ihre Vorzüge, abhängig vom Anwendungsfall. Raster-Tiles eignen sich für detaillierte Darstellungen und statische Karten-Stile, während Vector-Tiles bei dynamischer Gestaltung, Interaktivität und effizienter Datenübertragung herausragen. Mit dem Wissen über die Unterschiede dieser Tile-Typen, können Sie eine fundierte Entscheidung bei der Auswahl des passenden Tile-Formats für Ihre spezifischen Kartierungsanforderungen treffen.

## Die API/Library

Es gibt keine einzelne universelle Library: Sie können auswählen, welche Ihren Anforderungen am besten entspricht. Die zwei beliebtesten JavaScript-Bibliotheken zur Darstellung von OSM-Tiles sind:

* OpenLayers – leistungsfähig und bewährt

* Leaflet – leichtgewichtig und einfach zu erlernen

APIs sind ebenfalls für mobile Plattformen erhältlich. Zum Beispiel [Route-Me](https://github.com/route-me/route-me){: target=blank} (iOS) und [osmdroid](https://github.com/osmdroid/osmdroid){: target=blank} (Android).

## Die Lizenz

Im Gegensatz zu Daten kommerzieller Anbieter, ist OpenStreetMap ‘Open Data’. Die Karten-Daten stehen Ihnen kostenlos zur Verfügung, mit der Freiheit sie zu kopieren und zu modifizieren. OSM’s Lizenz ist die [Open Database Licence](http://opendatacommons.org/licenses/odbl/summary/){: target=blank}.

Ihre Verpflichtungen sind:

* Namensnennung. Sie müssen OpenStreetMap als Urheber mit der gleichen Sichtbarkeit angeben, die man auch bei der Verwendung eines kommerziellen Anbieters erwarten würde. Siehe [OSM’s copyright guidelines](http://www.openstreetmap.org/copyright){: target=blank}.

* Weitergabe unter gleichen Bedingungen (Share-Alike). Wenn Sie jegliche angepasste Version von OSM’s Karten-Daten verwenden, oder damit erstellte Arbeiten, dürfen Sie diese angepasste Datenbank ebenfalls nur unter der ODbL anbieten.
