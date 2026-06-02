---
layout: docs
title: Einen Tile-Server manuell erstellen (Debian 11)
dist: Debian 11
dl_timestamp: "2020-11-13T21:42:03Z"
lang: de
---

# {{ title }}

!!! info ""
    Diese Seite beschreibt die Installation, Einrichtung und Konfiguration aller notwendigen Software, um einen eigenen Tile-Server zu betreiben. Diese Schritt-für-Schritt-Anleitungen wurden für [Debian Linux](https://www.debian.org/){: target=_blank} 11 (bullseye), und wurden im November 2020 getestet.

## Software Installation

Der OSM Tile-Server-Stack ist eine Sammlung von Programmen und Bibliotheken, die zusammenarbeiten um einen Tile-Server zu erstellen. Wie so oft bei OpenStreetMap gibt es viele verschiedene Wege, dieses Ziel zu erreichen und fast alle Komponenten besitzen Alternativen mit verschiedenen speziellen Vor- und Nachteilen. Diese Anleitung beschreibt die üblichste Variante, die der auf den Haupt-OpenStreetMap.org-Tile-Servern eingesetzten ähnlich ist.

Sie besteht aus 5 Haupt-Komponenten: `mod_tile`, `renderd`, `mapnik`, `osm2pgsql` und einer `postgresql/postgis` Datenbank. Mod_tile ist ein Apache-Modul, dass zwischengespeicherte Tiles verteilt und entscheidet, welche Tiles erneut gerendert werden  müssen&nbsp;– entweder, weil sie noch gar nicht zwischengespeichert sind, oder weil sie nicht mehr aktuell sind. Renderd stellt ein Prioritäts-Warteschlangensystem für unterschiedliche Arten von Anfragen zur Verfügung, um die Last durch die Rendering-Anfragen zu managen und zu glätten. Mapnik wird von `renderd` verwendet und ist die Software-Bibliothek, die das tatsächliche Rendern durchführt.

Dank der Arbeit der Debian-Maintainer, die aktuellsten Versionen dieser Pakete in {{ dist }} zu integrieren, sind diese Anleitungen etwas kürzer als die vorherigen Versionen.

Diese Anleitungen wurden geschrieben und getestet für einen frisch installierten {{ dist }} Server. Falls Sie bereits andere Versionen mancher Programme installiert haben (weil Sie vielleicht von einer früheren Version aktualisiert haben oder einige PPAs eingerichtet haben, von denen geladen wird), müssen Sie eventuell einige Anpassungen vornehmen.

Um diese Komponenten zu bauen, muss zunächst eine Auswahl von Abhängigkeiten installiert werden.

Debian kommt standardmäßig nicht mit `sudo`, deshalb müssen wir uns als `root` einloggen um den ersten Teil durchzuführen:

```sh
--8<-- "docs/assets/serving-tiles/debian-deps.txt"
```

Während wir noch als root eingeloggt sind, stellen wir sicher, dass unser verwendetes Haupt-Nutzerkonto `sudo` zu root durchführen kann. Tauschen Sie in der Zeile unten `youruseraccount` gegen Ihr verwendetes Nutzerkonto aus. Versuchen sie nicht, alles folgenden als `root` auszuführen; Es wird nicht funktionieren.

```sh
usermod -aG sudo youruseraccount
exit
```

Das wechselt wieder zu Ihrem Nutzerkonto. Melden Sie sich ab und wieder an, und dann:

```sh
sudo whoami
```

Das sollte `root` ausgeben.

An diesem Punkt wurden ein paar neue Konten hinzugefügt. Mit `tail /etc/passwd` können Sie sie anzeigen. `postgres` wird zum Verwalten der Datenbank benutzt, in der wir die Daten zum Rendern bereithalten. `_renderd` wird für den `renderd`-Daemon gebraucht. Und wir müssen sicherstellen, dass viele der folgenden Kommandos mit diesem Benutzer ausgeführt werden.

Jetzt müssen Sie eine postgis-Datenbank erzeugen. Die Standardeinstellungen vieler Programme nehmen an, dass die Datenbank `gis` genannt wird und wir werden in dieser Anleitung die gleiche Konvention verwenden, obwohl dies nicht notwendig ist. Beachten Sie, dass `_renderd` im Folgenden mit dem Benutzer übereinstimmt, von dem aus der `renderd`-Daemon laufen wird.

```sh
sudo -u postgres -i
createuser _renderd
createdb -E UTF8 -O _renderd gis
```

Noch während Sie als Benutzer `postgres` arbeiten, richten Sie PostGIS auf der PostgreSQL Datenbank ein:

```sh
psql
```

(das wird Sie zu einem `postgres=#` Prompt bringen)

```sh
\c gis
```

(es wird ausgeben "You are now connected to database 'gis' as user 'postgres'".)

```sql
CREATE EXTENSION postgis;
```

(es wird ausgeben CREATE EXTENSION)

```sql
CREATE EXTENSION hstore;
```

(es wird ausgeben CREATE EXTENSION)

```sql
ALTER TABLE geometry_columns OWNER TO _renderd;
```

(es wird ausgeben ALTER TABLE)

```sql
ALTER TABLE spatial_ref_sys OWNER TO _renderd;
```

(es wird ausgeben ALTER TABLE)

```sh
\q
```

(es wird psql beenden und zu einem normalen Linux Prompt zurückkehren)

```sh
exit
```

(um zu dem Benutzer zurück zu kehren, der wir waren bevor wir `sudo -u postgres -i` weiter oben ausgeführt haben)

## Mapnik

Mapnik wurde oben installiert. Wir werden prüfen, dass es korrekt installiert wurde, indem wir folgendes ausführen:

```py
python3
>>> import mapnik
>>>
```

If python replies with the second chevron prompt `>>>` and without errors, then Mapnik library was found by Python. Congratulations! You can leave Python with this command:
Wenn Python mit dem zweiten Chevron Prompt `>>>` und ohne Fehlermeldungen antwortet, dann wurde die Mapnik-Bibliothek von Python gefunden. Glückwunsch! Sie können Python mit diesem Kommando verlassen:

```py
>>> quit()
```

## Stylesheet configuration

Now that all of the necessary software is installed, you will need to download and configure a stylesheet.

The style we'll use here is the one that use by the "standard" map on the openstreetmap.org website. It's chosen because it's well documented, and should work anywhere in the world (including in places with non-latin placenames). There are a couple of downsides though - it's very much a compromise designed to work globally, and it's quite complicated to understand and modify, should you need to do that.

The home of "OpenStreetMap Carto" on the web is <https://github.com/gravitystorm/openstreetmap-carto/>{: target=_blank}, and it has its own installation instructions at <https://github.com/gravitystorm/openstreetmap-carto/blob/master/INSTALL.md>{: target=_blank}, although we'll cover everything that needs to be done here.

Here we're assuming that we're storing the stylesheet details in a directory below `~/src` below the home directory of the whichever non-root account you are using:

```sh
mkdir ~/src
cd ~/src
git clone https://github.com/gravitystorm/openstreetmap-carto
cd openstreetmap-carto
git pull --all
git switch --detach v5.9.0
```

The "git switch" is needed because that's the latest release that you can see at OpenStreetMap, but OSM Carto is in the process of moving to a different database format. See OSM Carto's [INSTALL.md](https://github.com/gravitystorm/openstreetmap-carto/blob/master/INSTALL.md) for the newer version.

Next, we'll install a suitable version of the `carto` compiler.

```sh
sudo npm install -g carto
carto -v
```

That should respond with a number that is at least as high as:

```sh
1.2.0
```

Then we convert the carto project into something that Mapnik can understand:

```sh
carto project.mml > mapnik.xml
```

You now have a Mapnik XML stylesheet at `/home/youraccountname/src/openstreetmap-carto/mapnik.xml`.

## Loading data

Initially, we'll load only a small amount of test data. Other download locations are available, but `download.geofabrik.de` has a wide range of options. In this example we'll download the data for Azerbaijan, which is currently about 32Mb.

Browse to <https://download.geofabrik.de/asia/azerbaijan.html>{: target=_blank} and note the "This file was last modified" date (e.g. "{{ dl_timestamp }}"). We'll need that later if we want to update the database with people's subsequent changes to OpenStreetMap. Download it as follows:

```sh
mkdir ~/data
cd ~/data
wget https://download.geofabrik.de/asia/azerbaijan-latest.osm.pbf
```

The following command will insert the OpenStreetMap data you downloaded earlier into the database. This step is very disk I/O intensive; importing the full planet might take many hours, days or weeks depending on the hardware. For smaller extracts the import time is much faster accordingly, and you may need to experiment with different `-C` values to fit within your machine's available memory. Note that the `_renderd` user is used for this process.

```sh
sudo -u _renderd \
    osm2pgsql -d gis --create --slim  -G --hstore \
    --tag-transform-script \
        ~/src/openstreetmap-carto/openstreetmap-carto.lua \
    -C 2500 --number-processes 1 \
    -S ~/src/openstreetmap-carto/openstreetmap-carto.style \
    ~/data/azerbaijan-latest.osm.pbf
```

It's worth explaining a little bit about what those options mean:

`-d gis`
: The database to work with (`gis` used to be the default; now it must be specified).

`--create`
: Load data into an empty database rather than trying to append to an existing one.

`--slim`
: osm2pgsql can use different table layouts; `slim` tables works for rendering.

`-G`
: Determines how multipolygons are processed.

`--hstore`
: Allows tags for which there are no explicit database columns to be used for rendering.

`--tag-transform-script ~/src/openstreetmap-carto/openstreetmap-carto.lua`
: Defines the lua script used for tag processing. This an easy is a way to process OSM tags before the style itself processes them, making the style logic potentially much simpler.

`-C 2500`
: Allocate 2.5 Gb of memory to osm2pgsql to the import process. If you have less memory you could try a smaller number, and if the import process is killed because it runs out of memory you'll need to try a smaller number or a smaller OSM extract.

`--number-processes 1`
: Use 1 CPU. If you have more cores available you can use more.

`-S ~/src/openstreetmap-carto/openstreetmap-carto.style`
: Create the database columns in this file (actually these are unchanged from "openstreetmap-carto")

`~/data/azerbaijan-latest.osm.pbf`
: The final argument is the data file to load.

That command will complete with something like "Osm2pgsql took 238s overall".

### Creating indexes

Since version v5.3.0, some extra indexes now need to be [applied manually](https://github.com/gravitystorm/openstreetmap-carto/blob/master/CHANGELOG.md#v530---2021-01-28){: target=_blank}:

```sh
cd ~/src/openstreetmap-carto/
sudo -u _renderd psql -d gis -f indexes.sql
```

It should respond with `CREATE INDEX` 16 times.

## Database functions

In version 5.9.0 of “OSM Carto” (released October 2024), some functions need to be loaded into the database manually. These can be added / re-loaded at any point using:

```sh
cd ~/src/openstreetmap-carto/
sudo -u _renderd psql -d gis -f functions.sql
```

### Shapefile download

Although most of the data used to create the map is directly from the OpenStreetMap data file that you downloaded above, some shapefiles for things like low-zoom country boundaries are still needed. To download and index these, using the same account as we used previously:

```sh
cd ~/src/openstreetmap-carto/
mkdir data
sudo chown _renderd data
sudo -u _renderd scripts/get-external-data.py
```

This process involves a sizable download and may take some time - not much will appear on the screen when it is running. Some data will go directly into the database, and some will go into a “data” directory below “openstreetmap-carto”. If there is a problem here, then the Natural Earth data may have moved - look at [this issue](https://github.com/nvkelso/natural-earth-vector/issues/581#issuecomment-913988101){: target=_blank} and other issues at Natural Earth for more details. If you need to change the Natural Earth download location your copy of [this file](https://github.com/gravitystorm/openstreetmap-carto/blob/master/external-data.yml){: target=_blank} is the one to edit.

### Fonts

In version v5.6.0 and above of Carto, fonts need to be installed manually:

```sh
cd ~/src/openstreetmap-carto/
scripts/get-fonts.sh
```

Our test data area (Azerbaijan) was chosen both because it was a small area and because some place names in that region have names containing non-latin characters.

## Setting up your webserver

### Configure renderd

The config file for `renderd` on {{ dist }} is `/etc/renderd.conf`. Edit that with a text editor such as nano:

```sh
sudo nano /etc/renderd.conf
```

Add a section like the following at the end:

```ini
[s2o]
URI=/hot/
TILEDIR=/var/lib/mod_tile
XML=/home/accountname/src/openstreetmap-carto/mapnik.xml
HOST=localhost
TILESIZE=256
MAXZOOM=20
```

The location of the XML file `/home/accountname/src/openstreetmap-carto/mapnik.xml` will need to be changed to the actual location on your system. You can change `[s2o]` and `URI=/hot/` as well if you like. If you want to render more than one set of tiles from one server you can - just add another section like `[s2o]` with a different name referring to a different map style. If you want it to refer to a different database to the default `gis` you can, but that's out of the scope of this document. If you've only got 2Gb or so of memory, you'll also want to reduce `num_threads` to 2. `URI=/hot/` was chosen so that the tiles generated here can more easily be used in place of the HOT tile layer at OpenStreetMap.org. You can use something else here, but `/hot/` is as good as anything.

When this guide was first written, the version of Mapnik provided by Debian was 3.0, and the `plugins_dir` setting in the `[mapnik]` part of the file was `/usr/lib/mapnik/3.0/input`. At the time of writing that's changed to 3.1, and so the relevant value is `/usr/lib/mapnik/3.1/input`. It may change again in the future. If an error occurs when trying to render tiles such as this:

!!! warning ""
    An error occurred while loading the map layer 's2o': Could not create datasource for type: 'postgis' (no datasource plugin directories have been successfully registered) encountered during parsing of layer 'landcover-low-zoom'

then look in `/usr/lib/mapnik` and see what version directories there are, and also look in `/usr/lib/mapnik/(version)/input` to make sure that a file `postgis.input` exists there.

### Configuring Apache

```sh
sudo mkdir /var/lib/mod_tile
sudo chown _renderd /var/lib/mod_tile
```

After doing so, restart `renderd`:

```sh
sudo /etc/init.d/renderd restart
```

If you look at `/var/log/syslog`, you should see messages from the `renderd` service. There will initially be some font errors - don't worry about those for now. Next:

```sh
sudo /etc/init.d/apache2 restart
```

In `syslog` you should see a message like:

```log
Nov 14 14:24:55 servername apachectl[19119]: [Sat Nov 14 14:24:55.526717 2020] [tile:notice] [pid 19119:tid 140525098995008] Loading tile config s2o at /hot/ for zooms 0 - 20 from tile directory /var/lib/mod_tile with extension .png and mime type image/png
```

Next, point a web browser at `http://your.server.ip.address/index.html` (change `your.server.ip.address` to your actual server address). You should see "Apache2 Debian Default Page".

!!! tip
    If you don't know what IP address it will have been assigned, you can likely use `ifconfig` to find out – if the network configuration is not too complicated it'll probably be the `inet addr` that is not `127.0.0.1`.

If you're using a server at a hosting provider then it's likely that your server's internal address will be different to the external address that has been allocated to you, but that external IP address will have already been sent to you, and it'll probably be the one that you're accessing the server on currently.

Note that this is just the `http` (port 80) site – you'll need to do a little bit more Apache configuration if you want to enable `https`, but that's out of the scope of these instructions. However, if you use "Let's Encrypt" to issue certificates, then the process of setting that up can also configure the Apache HTTPS site as well.

Next, point a web browser at: `http://your.server.ip.address/hot/0/0/0.png`

You'll need to edit that of course if you changed `URI=/hot/` above. You should see a small map of the world. If you don't, investigate the errors that it displays. These will most likely be permissions errors, or perhaps related to accidentally missing some steps from the instructions above. If you don't get a tile and get other errors again, save the full output in a Pastebin and ask a question about the problem somewhere like [`community.openstreetmap.org`](https://community.openstreetmap.org){: target=_blank}.

## Viewing tiles

In order to see tiles, we’ll cheat and use an html file `sample_leaflet.html` that allows you to view a very simple map. To obtain this:

```sh
cd /var/www/html
sudo wget https://raw.githubusercontent.com/SomeoneElseOSM/mod_tile/switch2osm/extra/sample_leaflet.html
sudo nano sample_leaflet.html
```

Edit so that the IP address matches `your.server.address` rather than just saying `127.0.0.1`. That should allow you to access this server from others. Then browse to `http://your.server.address/sample_leaflet.html`.

The initial map display will take a little while. You'll be able to zoom in and out, but depending on server speed some tiles may initially display as grey because they can't be rendered in time for the browser. However, once done they’ll be ready for the next time that they are needed. If you look in `/var/log/syslog` you should see requests for tiles.

If desired, you can increase the setting `ModTileMissingRequestTimeout` in `/etc/apache2/conf-available/renderd.conf` from 10 seconds to perhaps 30 or 60, in order to wait longer for tiles to be rendered in the background before a grey tile is given to the user. Make sure you `#!sh sudo service renderd restart` and `#!sh sudo service apache2 restart` after changing it.

Congratulations. Head over to the [using tiles](/using-tiles/index.md) section to create a map that uses your new tile server.
