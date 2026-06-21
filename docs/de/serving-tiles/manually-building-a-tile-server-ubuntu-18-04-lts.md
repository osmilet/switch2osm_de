---
layout: docs
title: Einen Tile-Server manuell erstellen (18.04 LTS)
dist: Ubuntu 18.04
dl_timestamp: "2017-02-26T21:43:02Z"
lang: de
---

# {{ title }}

!!! info ""
    Diese Seite beschreibt die Installation, Einrichtung und Konfiguration aller notwendigen Software, um einen eigenen Tile-Server zu betreiben. Diese Schritt-für-Schritt-Anleitungen wurden für [Ubuntu Linux](https://ubuntu.com/){: target=_blank} 18.04 LTS (Bionic Beaver) geschrieben, und wurden im Mai 2020 getestet.

## Software Installation

Der OSM Tile-Server-Stack ist eine Sammlung von Programmen und Bibliotheken, die zusammenarbeiten um einen Tile-Server zu erstellen. Wie so oft bei OpenStreetMap gibt es viele verschiedene Wege, dieses Ziel zu erreichen und fast alle Komponenten besitzen Alternativen mit verschiedenen speziellen Vor- und Nachteilen. Diese Anleitung beschreibt die üblichste Variante, die der auf den Haupt-OpenStreetMap.org-Tile-Servern eingesetzten ähnlich ist.

Sie besteht aus 5 Haupt-Komponenten: `mod_tile`, `renderd`, `mapnik`, `osm2pgsql` und einer `postgresql/postgis` Datenbank. Mod_tile ist ein Apache-Modul, dass zwischengespeicherte Tiles verteilt und entscheidet, welche Tiles erneut gerendert werden  müssen&nbsp;– entweder, weil sie noch gar nicht zwischengespeichert sind, oder weil sie nicht mehr aktuell sind. Renderd stellt ein Priorität-Warteschlangensystem für unterschiedliche Arten von Anfragen zur Verfügung, um die Last durch die Rendering-Anfragen zu managen und zu glätten. Mapnik wird von `renderd` verwendet und ist die Software-Bibliothek, die das tatsächliche Rendern durchführt.

Beachten Sie, dass diese Anleitungen für einen frisch installierten {{ dist }} Server geschrieben und getestet wurden. Falls Sie bereits andere Versionen mancher Programme installiert haben (weil Sie vielleicht von einer früheren Ubuntu Version aktualisiert haben oder einige PPAs eingerichtet haben, von denen geladen wird) müssen Sie eventuell einige Anpassungen vornehmen.

Diese Anleitung nimmt an, dass Sie alles mit einem nicht-root Benutzer via `sudo` ausführen. Der nicht-root Benutzername, der unten standardmäßig verwendet wird, lautet `renderaccount` - Sie können diesen lokal erstellen, wenn Sie möchten, oder Skripte so anpassen, dass sie auf einen anderen Benutzernamen verweisen. Falls Sie den `renderaccount`-Benutzer erstellen, müssen Sie ihn zur Gruppe von Benutzern hinzufügen, die `sudo` zu root ausführen dürfen. Von Ihrem normalen nicht-root Benutzerkonto:

```sh
sudo -i
usermod -aG sudo renderaccount
exit
```

Um diese Komponenten zu bauen, muss zunächst eine Auswahl von Abhängigkeiten installiert werden:

```sh
--8<-- "docs/assets/serving-tiles/ubuntu-18-04-deps.txt"
```

Antworten Sie yes zum installieren. Das wird eine Weile dauern, also holen Sie sich eine Tasse Tee. Diese Liste enthält verschiedene Hilfsmittel und Bibliotheken, den Apache Webserver und "carto", welches benutzt wird um Carto-CSS Stylesheets in etwas zu konvertieren, das "mapnik", der Karten-Renderer, verstehen kann. Wenn dies fertig ist, installieren Sie den zweiten Satz von Voraussetzungen:

## postgresql / postgis installieren

Auf Ubuntu gibt es vor-gepackte Versionen von sowohl `postgis` als auch `postgresql`. Damit können sie einfach mit dem Ubuntu Paketmanager installiert werden.

```sh
sudo apt-get install \
    postgresql \
    postgresql-contrib \
    postgis \
    postgresql-10-postgis-2.4 \
    postgresql-10-postgis-scripts
```

`postgresql` stellt hier die Datenbank dar, in der wir die Karten-Daten speichern werden und `postgis` fügt etwas zusätzliche grafische Unterstützung hinzu. Antworten Sie erneut mit yes zum installieren.

Jetzt müssen Sie eine postgis-Datenbank erzeugen. Die Standardeinstellungen verschiedener Programme gehen davon aus, dass die Datenbank `gis` genannt wird und wir werden die gleiche Konvention in dieser Anleitung verwenden, obwohl es nicht notwendig ist. Setzen Sie ihren Benutzernamen für `renderaccount` ein, wo es im Folgenden verwendet wird. Dies sollte der Benutzername sein, der Karten mit Mapnik rendern wird.

```sh
sudo -u postgres -i
createuser renderaccount
createdb -E UTF8 -O renderaccount gis
```

Noch während Sie als Benutzer `postgres` arbeiten, richten Sie PostGIS auf der PostgreSQL Datenbank ein (setzen Sie wieder Ihren Benutzernamen für `renderaccount` unten ein):

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
ALTER TABLE geometry_columns OWNER TO renderaccount;
```

(es wird ausgeben ALTER TABLE)

```sql
ALTER TABLE spatial_ref_sys OWNER TO renderaccount;
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

Wenn Sie nicht bereits einen erstellt haben, erstellen Sie jetzt auch einen Unix-Benutzer für diesen Benutzer. Ein Passwort wählen, wenn dazu aufgefordert wird:

```sh
sudo useradd -m renderaccount
sudo passwd renderaccount
```

Ersetzen Sie obeb wieder `renderaccount` mit dem nicht-root Benutzer, den Sie gewählt haben.

### osm2pgsql installieren

Wir müssen ein paar Software-Teile aus ihren Quellen installieren. Das erste davon ist `osm2pgsql`. Es gibt verschiedene Tools für Import und Verwaltung von OpenStreetMap Daten in einer Datenbank. Hier werden wir `osm2pgsql` benutzen, welches vermutlich am beliebtesten ist.

```sh
mkdir ~/src
cd ~/src
git clone https://github.com/openstreetmap/osm2pgsql
cd osm2pgsql
```

Der von osm2pgsql verwendete Build-Mechanismus hat sich seit älteren Versionen verändert, daher werden wir dafür ein paar weitere Voraussetzungen installieren müssen:

```sh
--8<-- "docs/assets/serving-tiles/ubuntu-18-04-osm2pgsql-deps.txt"
```

Antworten Sie wieder mit yes zum installieren.

```sh
mkdir build && cd build
cmake ..
```

(die Ausgabe davon sollte mit "build files have been written to" abschließen)

```sh
make
```

(die Ausgabe davon sollte mit "[100%] Built target osm2pgsql" abschließen)

```sh
sudo make install
```

## Mapnik

Als nächstes installieren wir Mapnik. Wir verwenden die Standard-Version von {{ dist }}:

```sh
--8<-- "docs/assets/serving-tiles/ubuntu-18-04-mapnik-deps.txt"
```

Wir überprüfen, dass Mapnik korrekt installiert wurde:

```py
python
>>> import mapnik
>>>
```

Wenn Python mit dem zweiten Chevron Prompt >>> und ohne Fehlermeldungen antwortet, dann wurde die Mapnik-Bibliothek von Python gefunden. Glückwunsch! Sie können Python mit diesem Kommando verlassen:

```py
>>> quit()
```

## mod_tile und renderd installieren

Als nächstes installieren wir mod_tile und renderd. `mod_tile` ist ein Apache-Modul, das Anfragen für Tiles verarbeitet; `renderd` ist ein Dienst, der Tiles tatsächlich rendert, wenn `mod_tile` sie anfragt. Wir werden den `switch2osm` Branch von <https://github.com/SomeoneElseOSM/mod_tile>{: target=_blank} verwenden, welcher wiederum von <https://github.com/openstreetmap/mod_tile>{: target=_blank} geforked ist, aber so modifiziert dass es {{ dist }} unterstützt und mit ein paar weiteren Veränderungen auch auf einem Standard Ubuntu Server läuft statt auf einem von OSM's Rendering-Servern.

### Den mod_tile Quellcode kompilieren

```sh
cd ~/src
git clone -b switch2osm git://github.com/SomeoneElseOSM/mod_tile.git
cd mod_tile
./autogen.sh
```

(das sollte mit `autoreconf: Leaving directory '.'` beenden.)

```sh
./configure
```

(das sollte mit `config.status: executing libtool commands` beenden)

```sh
make
```

Beachten Sie, dass hierbei einige "beunruhigende" Meldungen über den Schirm laufen werden. Es sollte jedoch mit "make[1]: Leaving directory '/home/renderaccount/src/mod_tile'" abschließen.

```sh
sudo make install
```

(das sollte mit "make[1]: Leaving directory '/home/renderaccount/src/mod_tile'" beenden)

```sh
sudo make install-mod_tile
```

(das sollte mit "chmod 644 /usr/lib/apache2/modules/mod_tile.so" beenden)

```sh
sudo ldconfig
```

(das sollte gar nichts ausgeben)

## Stylesheet-Konfiguration

Jetzt, da alle notwendige Software installiert ist, müssen sie ein Stylesheet (Formatvorlage) herunterladen und konfigurieren.

Der Stil, den wir hier verwenden werden, ist derjenige, der auch von der "Standard"-Karte auf der openstreetmap.org Website verwendet wird. Er wird gewählt, da er gut dokumentiert ist und überall auf der Welt funktionieren sollte (inklusive an Orten mit nicht-lateinischen Ortsnamen). Es gibt jedoch auch einige Nachteile - es ist ein Kompromiss mit dem Ziel global zu funktionieren und es ist ziemlich kompliziert zu verstehen und zu modifizieren, sollten Sie den Bedarf haben dies zu tun.

Das Zuhause von "OpenStreetMap Carto" im Internet ist <https://github.com/gravitystorm/openstreetmap-carto/>{: target=_blank} und es hat seine eigene Installationsanleitung bei <https://github.com/gravitystorm/openstreetmap-carto/blob/master/INSTALL.md>{: target=_blank}, obwohl wir alles notwendige hier behandeln werden.

Hier nehmen wir an, dass wir die Stylesheet-Details in einem Verzeichnis unterhalb von `~/src` unterhalb des Home-Verzeichnis des `renderaccount`-Benutzers speichern (oder welchen anderen Benutzer Sie verwenden)

```sh
cd ~/src
git clone https://github.com/gravitystorm/openstreetmap-carto
cd openstreetmap-carto
```

Als nächstes werden wir eine passende Version des `carto` Compilers installieren. Dieser ist neuer als die Version, die mit Ubuntu kommt, daher müssen wir dies ausführen:

```sh
sudo apt-get install nodejs-dev node-gyp libssl1.0-dev
```

Verwirrenderweise wird dies einige zuvor installierte Dinge entfernen. Dann machen Sie:

```sh
sudo apt install npm nodejs
sudo npm install -g carto
carto -v
```

Dies sollte mit einer Nummer antworten, die mindestens so hoch ist wie:

```sh
carto 1.2.0 (Carto map stylesheet compiler)
```

Dann konvertieren wir das carto-Projekt in etwas, das Mapnik verstehen kann:

```sh
carto project.mml > mapnik.xml
```

Sie haben jetzt ein Mapnik XML Stylesheet in `/home/renderaccount/src/openstreetmap-carto/mapnik.xml`.

## Daten laden

Initial laden wir nur eine kleine Menge von Test-Daten. Andere Download-Quellen sind auch verfügbar, aber `download.geofabrik.de` hat eine große Auswahl an Optionen. In diesem Beispiel werden wir die Daten für Aserbaidschan herunterladen, was zur Zeit etwa 32Mb sind.

Navigieren Sie zu <https://download.geofabrik.de/asia/azerbaijan.html>{: target=_blank} und notieren sich das Datum von "This file was last modified" (z.B. "{{ dl_timestamp }}"). Wir werden es später benötigen, wenn wir die Datenbank mit Änderungen aktualisieren wollen, die Menschen später an OpenStreetMap vorgenommen haben. Laden Sie es folgendermaßen herunter:

```sh
mkdir ~/data
cd ~/data
wget https://download.geofabrik.de/asia/azerbaijan-latest.osm.pbf
```

Das folgende Kommando wird die vorher heruntergeladenen OpenStreetMap-Daten zur Datenbank hinzufügen. Dieser Schritt ist sehr anspruchsvoll bei den Laufwerkszugriffen; Abhängig von der Hardware, könnte es mehrere Stunden, Tage oder Wochen dauern, den kompletten Planeten zu importieren. 
Für kleinere Auszüge ist die Import-Zeit dementsprechend deutlich kürzer und Sie müssen möglicherweise mit verschiedenen Werten für `-C` experimentieren um es an den verfügbaren Speicher ihres Rechners anzupassen.

```sh
osm2pgsql -d gis --create --slim  -G --hstore \
    --tag-transform-script \
        ~/src/openstreetmap-carto/openstreetmap-carto.lua \
    -C 2500 --number-processes 1 \
    -S ~/src/openstreetmap-carto/openstreetmap-carto.style \
    ~/data/azerbaijan-latest.osm.pbf
```

Es lohnt sich ein wenig zu erklären, was diese Optionen bedeuten:

`-d gis`
: Die Datenbank, mit der gearbeitet werden soll (`gis` wurde in der Vergangenheit standardmäßig verwendet; jetzt muss es explizit spezifiziert werden).

`--create`
: Lade Daten in eine leere Datenbank anstatt zu versuchen sie an eine existierende anzuhängen.

`--slim`
: osm2pgsql kann verschiedene Tabellenformate verwenden; `slim` Tabellen sind für das Rendern geeignet.

`-G`
: Legt fest, wie Multipolygone verarbeitet werden.

`--hstore`
: Erlaubt, dass Tags zum Rendern benutzt werden, für die keine expliziten Datenbank-Spalten bestehen.

`--tag-transform-script ~/src/openstreetmap-carto/openstreetmap-carto.lua`
: Definiert das lua Script, das zur Tag-Verarbeitung verwendet wird. Dies ist ein einfacher Weg, um OSM-Tags zu verarbeiten, bevor sie der Stil selbst verarbeitet. Dadurch kann die Stil-Logik möglicherweise einfacher gehalten werden.

`-C 2500`
: Vergebe 2.5 Gb an Speicher zu osm2pgsql für den Import-Prozess. Wenn Sie weniger Speicher zur Verfügung haben, können Sie eine kleinere Zahl versuchen. Und wenn der Import-Prozess wegen zu wenig Speicher abgebrochen wird, müssen Sie eine kleinere Zahl probieren oder ein kleineres OSM-Extrakt verwendet.

`--number-processes 1`
: Benutze 1 CPU. Wenn Sie mehr Kerne zur Verfügung haben, können Sie mehr verwenden.

`-S ~/src/openstreetmap-carto/openstreetmap-carto.style`
: Erzeuge die Datenbank-Spalten in dieser Datei (tatsächlich sind diese unverändert zu "openstreetmap-carto")

`~/data/azerbaijan-latest.osm.pbf`
: Das finale Argument ist die Datei mit den zu ladenden Daten.

Dieses Kommando wird mit etwas wie "Osm2pgsql took 238s overall" abschließen.

### Shapefile herunterladen

Obwohl die meisten Daten, die zur Kartenerstellung benötigt werden, direkt aus der zuvor heruntergeladenen Datei mit OpenStreetMap-Daten kommen, sind noch einige Shapefiles nötig für Dinge wie Länder-Grenzen bei niedrigen Zoom-Stufen. Um diese herunterzuladen und zu indizieren:

```sh
cd ~/src/openstreetmap-carto/
scripts/get-external-data.py
```

Dieser Prozess beinhaltet einen beträchtlich Download und könnte einige Zeit dauern – während es läuft, wird auf dem Bildschirm nicht viel erscheinen. Tatsächlich wird es aber ein "data"-Verzeichnis unterhalb von  "openstreetmap-carto" befüllen.

### Schriftarten

In Version v5.6.0 und höher von Carto müssen Schriftarten manuell installiert werden:

```sh
cd ~/src/openstreetmap-carto/
scripts/get-fonts.sh
```

Unsere Test-Daten-Region (Aserbaidschan) wurde gewählt, weil es ein kleines Gebiet ist, und auch, weil einige Ortsbezeichnungen in dieser Region Namen mit nicht-lateinischen Schriftzeichen besitzen.

## Ihren Webserver einrichten

### renderd konfigurieren

Die Konfigurations-Datei für `renderd` ist `/usr/local/etc/renderd.conf`. Editieren Sie diese mit einem Texteditor wie beispielsweise nano:

```sh
sudo nano /usr/local/etc/renderd.conf
```

Ein paar Zeilen müssen hier geändert werden. Im `renderd`-Abschnitt:

```ini
num_threads=4
```

Wenn Sie nur ungefähr 2Gb Speicher zur Verfügung haben, dann werden dies auf 2 reduzieren wollen. Der `ajt`-Abschnitt entspricht einem "benannten Karten-Stil" namens `ajt`. Wenn Sie wollen,können Sie mehr als einen dieser Abschnitte haben, vorausgesetzt, dass die URI für jeden unterschiedlich ist. Die `XML`-Zeile wird geändert werden müssen, zu etwas wie:

```ini
XML=/home/renderaccount/src/openstreetmap-carto/mapnik.xml
```

Sie werden `renderaccount` in den nicht-root Benutzernamen ändern wollen, den Sie oben benutzt haben.

```ini
URI=/hot/
```

Das wurde gewählt, so dass die hier erzeugten Tiles einfacher an der Stelle der HOT Tile-Ebene auf OpenStreetMap.org verwendet werden können. Sie können hier etwas beliebiges anderes wählen, aber `/hot/` ist genauso gut wie alles andere.

### Apache konfigurieren

```sh
sudo mkdir /var/lib/mod_tile
sudo chown renderaccount /var/lib/mod_tile
```

```sh
sudo mkdir /var/run/renderd
sudo chown renderaccount /var/run/renderd
```

Jetzt müssen wir Apache mit `mod_tile` bekannt machen, also mit nano (oder einem anderen Editor):

```sh
sudo nano /etc/apache2/conf-available/mod_tile.conf
```

Fügen Sie die folgende Zeile zu dieser Datei hinzu:

```ini
LoadModule tile_module /usr/lib/apache2/modules/mod_tile.so
```

Speichern Sie es und führen Sie dann aus:

```sh
sudo a2enconf mod_tile
```

Das wird ausgeben, dass Sie `service apache2 reload` ausführen sollen um die neue Konfiguration zu aktivieren; Wir werden das jetzt noch nicht tun.

Jetzt müssen wir Apache mit `renderd` bekannt machen. Mit nano (oder einem anderen Editor):

```sh
sudo nano /etc/apache2/sites-available/000-default.conf
```

Fügen Sie das folgenden zwischen den `ServerAdmin`- und `DocumentRoot`-Zeilen hinzu:

```ini
LoadTileConfigFile /usr/local/etc/renderd.conf
ModTileRenderdSocketName /var/run/renderd/renderd.sock
# Timeout before giving up for a tile to be rendered
ModTileRequestTimeout 0
# Timeout before giving up for a tile to be rendered that is otherwise missing
ModTileMissingRequestTimeout 30
```

Und Apache zweimal neu laden:

```sh
sudo service apache2 reload
sudo service apache2 reload
```

(Ich vermute es muss zwei mal gemacht werden, weil Apache "verwirrt" ist, wenn es im Betrieb neu konfiguriert wird)

Wenn Sie mit einem Webbrowser `http://your.server.ip.address/index.html` besuchen, sollten Sie die "It works!"-Seite Ubuntu / Apache bekommen.

!!! tip
    Falls Sie nicht wissen, welcher IP-Adresse es zugewiesen wurde, dann können Sie voraussichtlich `ifconfig` benutzen, um es herauszufinden – wenn die Netzwerkkonfiguration nicht zu kompliziert ist, dann ist es wahrscheinlich die `inet addr`, die nicht `127.0.0.1` ist.

Wenn Sie einen Server bei einem Hosting-Anbieter verwenden, dann ist es wahrscheinlich, dass die interne Adresse Ihres Servers sich von der externen Adresse unterscheidet, die Ihnen zugewiesen wurde. Aber diese externe Adresse wird Ihnen bereits übermittelt worden sein und wird vermutlich die sein, über die Sie bereits auf den Server zugreifen.

Beachten Sie, dass dies nur die `http`-Seite (Port 80) ist – Sie werden etwas mehr Apache Konfiguration betreiben müssen, wenn sie `https` aktivieren wollen, aber das ist nicht mehr Teil dieser Anleitung. Wenn Sie jedoch "Let's Encrypt" zur Ausstellung von Zertifikaten benutzen, dann kann der Prozess dies einzurichten auch die Konfiguration der Apache HTTPS-Seite beinhalten.

### renderd zum ersten mal ausführen

Als nächstes weden wir `renderd` ausführen, um zu versuchen ein paar Tiles zu rendern. Wir führen es zunächst im Vordergrund aus, damit wir alle Fehler sehen können, sobald sie auftreten:

```sh
renderd -f -c /usr/local/etc/renderd.conf
```

Sie könnten hier einige Warnungen sehen - machen Sie sich deswegen zunächst keine Sorgen. Sie sollten keine Fehlermeldungen erhalten. Falls doch, speichern Sie die komplette Ausgabe in einen Pastebin und stellen eine Frage zu dem Problem an einem Ort wie [`community.openstreetmap.org`](https://community.openstreetmap.org){: target=_blank} (wenn Sie zum Pastebin verlinken, fügen Sie nicht den ganzen Text aus der Frage ein).

Besuchen Sie mit einem Webbrowser: `http://yourserveripaddress/hot/0/0/0.png`

Sie sollten in ihrem Browser eine Karte der Welt sehen und weitere Debug-Ausgaben auf der Komandozeile, inklusive "DEBUG: START TILE" und "DEBUG: DONE TILE". Ignorieren Sie alle "DEBUG: Failed to read cmd on fd"-Meldungen - das ist kein Fehler. Wenn Sie kein Tile erhalten und andere Fehler bekommen, speichern Sie die komplette Ausgabe in einen Pastebin und stellen eine Frage zu dem Problem an einem Ort wie [`community.openstreetmap.org`](https://community.openstreetmap.org){: target=_blank}.

Wenn dies alles funktioniert, drücken Sie ++strg+c++ um den Vordergrund-Render-Prozess zu stoppen.

### renderd im Hintergrund ausführen

Als nächstes richten wir `renderd` ein um im Hintergrund zu laufen. Editieren Sie zuerst die Datei `~/src/mod_tile/debian/renderd.init` so, dass "RUNASUSER" auf den nicht-Root-Benutzer gesetzt ist, den Sie zuvor benutzt haben, wie `renderaccount`, dann kopieren Sie die Datei in das system-Verzeichnis:

```sh
nano ~/src/mod_tile/debian/renderd.init
sudo cp ~/src/mod_tile/debian/renderd.init /etc/init.d/renderd
sudo chmod u+x /etc/init.d/renderd
sudo cp ~/src/mod_tile/debian/renderd.service /lib/systemd/system/
```

Die `renderd.service` Datei ist eine `systemd`-service Datei. Die hier verwendete Version ruft einfach klassische init-Kommandos auf. Um zu testen, dass das Start-Kommando funktioniert:

```sh
sudo /etc/init.d/renderd start
```

(das sollte mit "[ ok ] Starting renderd (via systemctl): renderd.service" antworten.)

Um es jedes mal automatisch starten zu lassen:

```sh
sudo systemctl enable renderd
```

## Tiles anzeigen

Um Tiles zu sehen werden wir schummeln und eine html-Datei `sample_leaflet.html` in mod_tile “extras”-Verzeichnis verwenden. Öffnen Sie die Datei einfach in einem Webbrowser auf dem auf dem System, auf dem Sie den Tile-Server installiert haben. Falls das nicht möglich ist, weil Sie auf einem Server ohne lokalen Webbrowser installiere, dann können Sie in der Datei `127.0.0.1` mit der IP-Adresse des Servers ersetzen und die Datei unterhalb von `/var/www/html` auf diesen Server kopieren.

Machen Sie von einer ssh-Verbindug aus:

```sh
tail -f /var/log/syslog | grep " TILE "
```

(beachten Sie die Leerzeichen um `" TILE "`)

Das wird eine jedes mal eine Zeile ausgeben wenn ein Tile angefordert wird und wenn das Rendering eines Tiles abgeschlossen ist.

Wenn Sie die Seite laden, sollten Sie einige Tile-Anfragen sehen. Zoomen Sie nach und nach raus. Sie werden in der ssh-Verbindung Anfragen für neue Tiles auftauchen sehen. Einige Tiles in niedriger Zoom-Stufe könnten eine längere Zeit (mehrere Minuten) zum erstmaligen Rendern benötigen, aber sobald sie fertig sind, werden sie für das nächsten mal, wenn sie benötigt werden, bereit sein.

Glückwunsch! Schauen Sie in die [Tiles verwenden](/using-tiles/index.md)-Bereich um eine Karte zu erstellen, die Ihren neuen Tile-Server verwendet.
