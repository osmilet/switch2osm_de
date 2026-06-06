---
layout: docs
title: Einen Docker Container verwenden
lang: de
---

# {{ title }}

Wenn Sie nur etwas ausprobieren wollen oder ein anderes OS als Ubuntu verwenden, und Sie benutzen Docker zur Containerisierung, dann können Sie [dies](https://github.com/Overv/openstreetmap-tile-server){: target=_blank} versuchen (Danke an alle dortigen Mitwirkenden). Es basiert auf der Anleitung [hier](/serving-tiles/manually-building-a-tile-server-ubuntu-22-04-lts.md), stellt aber einen vorgefertigten Container dar, den Sie installieren können.

## Docker

Falls Die Docker nicht bereits installiert haben, gibt es eine Menge von "How-Tos" - beispielsweise [hier](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-debian-10){: target=_blank}.

Sie werden ungefähr 30GB Speicherplatz selbst für ein kleines Daten-Extrakt benötigen, da die weltweiten Grenz-Daten, die zur Datenbank hinzugefügt werden, bereits ziemlich groß sind.

## OpenStreetMap Daten

In diesem Beispiel-Durchlauf werde ich Daten für Sambia herunterladen und importieren, aber jede OSM .pbf-Datei sollte funktionieren. Zum Testen probieren Sie zu erst ein kleines .pbf. Eingeloggt als der nicht-root-Benutzer, mit dem Sie Docker betreiben, laden Sie die Daten für Sambia herunter:

```sh
cd
wget https://download.geofabrik.de/africa/zambia-latest.osm.pbf
```

Erzeugen Sie ein Volume für die Daten:

```sh
docker volume create osm-data
```

Installieren und importieren Sie die Daten:

```sh 
time \
    docker run \
    -v /home/renderaccount/zambia-latest.osm.pbf:/data/region.osm.pbf \
    -v osm-data:/data/database/ \
    overv/openstreetmap-tile-server import
```

Der Pfad zur Daten-Datei muss der absolute Pfad zur Daten-Datei sein - es darf kein relativer Pfad sein. In diesem Beispiel befindet sie sich im root-Verzeichnis des Benutzers "renderaccount".  Wenn etwas fehlschlägt, werden Sie "docker volume rm osm-data" ausführen und ab "docker volume create osm-data" oben neu beginnen müssen. Am Ende des Vorgangs sollten Sie dies sehen:

```sh
INFO:root: Import complete
```

Falls Sie etwas sehen wie:

```sh
/data/region.osm.pbf: Is a directory
```

oder

```sh
createuser: error: creation of new role failed: ERROR: role "renderer" already exists
```

dann ist etwas schief gelaufen; Sie werden "docker ps -a" benutzen müssen, um den fehlgeschlagenen Container zu identifizieren; "docker rm" (gefolgt von der Container-ID) um ihn zu löschen. Und dann löschen und erzeugen Sie "osm-data" wie oben beschrieben neu.

Wie lange dies dauert hängt stark von der lokalen Netzwerkgeschwindigkeit und der Größe des Gebiets ab, das Sie laden. Das in diesem Beispiel verwendete Sambia ist vergleichsweise klein.

Beachten Sie, dass die Fehlermeldungen etwas kryptisch sein können, wenn etwas fehlschlägt, und unglücklicherweise kann der Import-Vorgang nach einem Fehler nicht fortgesetzt werden. Beachten Sie ebenfalls, dass neuere Versionen des Docker-Containers neuere Versionen von postgres verwenden könnten, somit könnte ein “osm-data”, das mit einer früheren Version erstellt wurde, nicht funktionieren - Möglicherweise müssen Sie es mit “docker volume rm osm-data” löschen und neu erstellen.

Für mehr Details, was es tatsächlich tut, sehen Sie sich [diese Datei](https://github.com/Overv/openstreetmap-tile-server/blob/master/Dockerfile){: target=_blank} an. Sie werden feststellen, dass es sehr ähnlich zu “Einen Tile-Server manuell erstellen”-Anleitung [hier](/serving-tiles/manually-building-a-tile-server-ubuntu-22-04-lts.md) ist, mit einigen kleineren Änderungen wie der Tile-URL und dem intern verwendeten Benutzerkonto. Wie Sie sehen, benutzt es im Inneren Ubuntu 22.04. Sie brauchen nicht direkt damit interagieren, aber Sie könnten, falls Sie wollten (via "docker exec -it mycontainernumber bash").

Um den Tile-Server ans Laufen zu bringen:

```sh
docker run \
    -p 8080:80 \
    -v osm-data:/data/database \
    -d overv/openstreetmap-tile-server \
    run
```

und um zu prüfen, dass es funktioniert, navigieren Sie aus einem Inkognito-Fenster zu:

`http://your.server.ip.address:8080/tile/0/0/0.png`

Sie sollten eine Karte der Welt in Ihrem Browser sehen. Dann versuchen Sie:

`http://your.server.ip.address:8080`

für eine Karte bei der Sie rein- und rauszoomen können. Tiles (insbesondere bei niedriger Zoom-Stufe) werden einen kurzen Moment zum erscheinen benötigen.

### Mehr Informationen

Tatsächlich unterstützt dieser Docker-Container eine Menge mehr als dieses einfache Beispiel hier - sehen Sie sich die [readme](https://github.com/Overv/openstreetmap-tile-server/blob/master/README.md){: target=_blank} an für mehr Details über Updates, Leistungsoptimierungen, etc.

### Tiles anzeigen

Für eine einfache “slippy map”, die Sie modifizieren können, können Sie eine html-Datei “sample_leaflet.html” verwenden, welche sich [hier](https://github.com/SomeoneElseOSM/mod_tile/blob/switch2osm/extra/sample_leaflet.html){: target=_blank} im “extra” Ordner von mod_tile befindet. Ändern Sie “hot” in der URL innerhalb der Datei zu “tile” und öffnen Sie die Datei dann einfach in einem Webbrowser auf Ihrem Rechner auf dem Sie den Docker-Container installiert haben. Falls das nicht möglich ist, weil Sie auf einem Server ohne lokalen Webbrowser installieren, dann müssen Sie in der Datei “127.0.0.1” mit der IP-Adresse des Servers ersetzen und die Datei unterhalb von “/var/www/html” auf diesen Server kopieren.

Wenn Sie ein anderes Gebiet laden möchten, wiederholen Sie den Prozess einfach ab “wget” oben. Leider ist es notwendig, jedes mal wenn Sie neue Daten laden wollen, "osm-data" zu löschen und neu zu erstellen.
