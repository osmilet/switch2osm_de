---
layout: docs
title: Einen Tile-Server manuell erstellen (Debian 13)
dist: Debian 13
dl_timestamp: "2025-08-15T12:40:00Z"
lang: de
---

# {{ title }}

!!! info ""
    Diese Seite beschreibt die Installation, Einrichtung und Konfiguration aller notwendigen Software, um einen eigenen Tile-Server zu betreiben. Diese Schritt-für-Schritt-Anleitungen wurden für [Debian Linux](https://www.debian.org/){: target=_blank} 13 (trixie) geschrieben und wurden im August 2025 getestet.

## Software Installation

Der OSM Tile-Server-Stack ist eine Sammlung von Programmen und Bibliotheken, die zusammenarbeiten um einen Tile-Server zu erstellen. Wie so oft bei OpenStreetMap gibt es viele verschiedene Wege, dieses Ziel zu erreichen und fast alle Komponenten besitzen Alternativen mit verschiedenen speziellen Vor- und Nachteilen. Diese Anleitung beschreibt die üblichste Variante, die der auf den Haupt-OpenStreetMap.org-Tile-Servern eingesetzten ähnlich ist.

Sie besteht aus 5 Haupt-Komponenten: `mod_tile`, `renderd`, `mapnik`, `osm2pgsql` und einer `postgresql/postgis` Datenbank. Mod_tile ist ein Apache-Modul, dass zwischengespeicherte Tiles verteilt und entscheidet, welche Tiles erneut gerendert werden  müssen&nbsp;– entweder, weil sie noch gar nicht zwischengespeichert sind, oder weil sie nicht mehr aktuell sind. Renderd stellt ein Prioritäts-Warteschlangensystem für unterschiedliche Arten von Anfragen zur Verfügung, um die Last durch die Rendering-Anfragen zu managen und zu glätten. Mapnik wird von `renderd` verwendet und ist die Software-Bibliothek, die das tatsächliche Rendern durchführt.

Diese Anleitungen wurden geschrieben und getestet für einen frisch installierten {{ dist }} Server. Falls Sie bereits andere Versionen mancher Programme installiert haben (weil Sie vielleicht von einer früheren Version aktualisiert haben oder einige PPAs eingerichtet haben, von denen geladen wird), müssen Sie eventuell einige Anpassungen vornehmen.

Um diese Komponenten zu bauen, muss zunächst eine Auswahl von Abhängigkeiten installiert werden.

Debian kommt möglicherweise nicht standardmäßig mit `sudo`, deshalb müssen wir uns eventuell als `root` einloggen, um den ersten Teil durchzuführen:

```sh
--8<-- "docs/assets/serving-tiles/debian13-deps.txt"
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

```sh
mapnik-render --version
```

Es sollte mit der Versionsnummer von Mapnik antworten:

```sh
version 4.0.7
```

## Stylesheet-Konfiguration

Jetzt, da alle notwendige Software installiert ist, müssen sie ein Stylesheet (Formatvorlage) herunterladen und konfigurieren.

Der Stil, den wir hier verwenden werden, ist derjenige, der auch von der "Standard"-Karte auf der openstreetmap.org Website verwendet wird. Er wird gewählt, da er gut dokumentiert ist und überall auf der Welt funktionieren sollte (inklusive an Orten mit nicht-lateinischen Ortsnamen). Es gibt jedoch auch einige Nachteile - es ist ein Kompromiss mit dem Ziel global zu funktionieren und es ist ziemlich kompliziert zu verstehen und zu modifizieren, sollten Sie den Bedarf haben dies zu tun.

Das Zuhause von "OpenStreetMap Carto" im Internet ist <https://github.com/gravitystorm/openstreetmap-carto/>{: target=_blank} und es hat seine eigene Installationsanleitung bei <https://github.com/gravitystorm/openstreetmap-carto/blob/master/INSTALL.md>{: target=_blank}, obwohl wir alles notwendige hier behandeln werden.

Hier nehmen wir an, dass wir die Stylesheet-Details in einem Verzeichnis unterhalb von `~/src` unterhalb des Home-Verzeichnis Ihres verwendeten nicht-root Benutzers speichern:

```sh
mkdir ~/src
cd ~/src
git clone https://github.com/gravitystorm/openstreetmap-carto
cd openstreetmap-carto
git pull --all
git switch --detach v5.9.0
```

Das "git switch" wird benötigt, weil dies das neueste Release ist, dass man auf OpenStreetMap sehen kann. Tatsächlich hat OSM Carto auf ein anderes Datenbankformat gewechselt. Siehe OSM Carto's [INSTALL.md](https://github.com/gravitystorm/openstreetmap-carto/blob/master/INSTALL.md) für die neuere Version. Das alte Format nennt sich `pgsql`, das neue `flex`; Später werden wir dazu eine [Warnung](https://osm2pgsql.org/doc/faq.html#the-pgsql-output-is-deprecated-what-does-that-mean) von `osm2pgsql` erhalten.

Als nächstes werden wir überprüfen, dass wir eine passende Version des `carto` Compilers installiert haben.

```sh
carto -v
```

Dies sollte mit einer Nummer antworten, die mindestens so hoch ist wie:

```log
1.2.0
```

Dann konvertieren wir das carto-Projekt in etwas, das Mapnik verstehen kann:

```sh
carto project.mml > mapnik.xml
```

Sie haben jetzt ein Mapnik XML Stylesheet in `/home/youraccountname/src/openstreetmap-carto/mapnik.xml`.

## Daten laden

Initial laden wir nur eine kleine Menge von Test-Daten. Andere Download-Quellen sind auch verfügbar, aber `download.geofabrik.de` hat eine große Auswahl an Optionen. In diesem Beispiel werden wir die Daten für Aserbaidschan herunterladen, was zur Zeit etwa 32Mb sind.

Navigieren Sie zu <https://download.geofabrik.de/asia/azerbaijan.html>{: target=_blank} und notieren sich das Datum von "This file was last modified" (z.B. "{{ dl_timestamp }}"). Wir werden es später benötigen, wenn wir die Datenbank mit Änderungen aktualisieren wollen, die Menschen später an OpenStreetMap vorgenommen haben. Laden Sie es folgendermaßen herunter:

```sh
mkdir ~/data
cd ~/data
wget https://download.geofabrik.de/asia/azerbaijan-latest.osm.pbf
```

Als nächstes müssen wir sicherstellen, dass der `_renderd`-Benutzer auf das Stylesheet zugreifen kann. Dafür benötigt er Zugriff darauf, wohin Sie es heruntergeladen haben. Und standardmäßig wird es keinen Zugriff auf Ihr Home-Verzeichnis haben. Wenn es in sich in `src` unterhalb Ihres Benutzerkontos befindet, dann wird

```sh
chmod o+rx ~
```

funktionieren. Falls Sie dies nicht tun möchten, dann können Sie es verschieben und Referenzen zum Speicherort der Datei in nachfolgenden Kommandos anpassen.

Das folgende Kommando wird die vorher heruntergeladenen OpenStreetMap-Daten zur Datenbank hinzufügen. Dieser Schritt ist sehr anspruchsvoll bei den Laufwerkszugriffen; Abhängig von der Hardware, könnte es mehrere Stunden, Tage oder Wochen dauern, den kompletten Planeten zu importieren. Für kleinere Auszüge ist die Import-Zeit dementsprechend deutlich kürzer und Sie müssen möglicherweise mit verschiedenen Werten für `-C` experimentieren um es an den verfügbaren Speicher ihres Rechners anzupassen. Beachten Sie, dass der `_renderd` Benutzer für diesen Prozess verwendet wird.

```sh
sudo -u _renderd \
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

Während Sie dies ausführen, werden Sie die oben erwähnte [Warnung](https://osm2pgsql.org/doc/faq.html#the-pgsql-output-is-deprecated-what-does-that-mean) erhalten.  Die aktuellste, sich in Entwicklung befindende, Version von "OSM Carto" wurde dazu geändert die "flex"-Ausgabe zu nutzen, aber die Version auf der OpenStreetMap-Webseite (welche diese Anleitung wiederspiegeln soll) tut die noch nicht.

Dieses Kommando wird mit etwas wie "Osm2pgsql took 238s overall" abschließen.

### Indizes erstellen

Einige Extra-Indizes müssen jetzt [manuell angewendet](https://github.com/gravitystorm/openstreetmap-carto/blob/master/CHANGELOG.md#v530---2021-01-28){: target=_blank} werden:

```sh
cd ~/src/openstreetmap-carto/
sudo -u _renderd psql -d gis -f indexes.sql
```

Es sollte 16 mal mit `CREATE INDEX` antworten.

## Datenbank-Funktionen

In Version 5.9.0 von “OSM Carto” (veröffentlicht Oktober 2024) müssen einige Funktionen manuell in die Datenbank geladen werden. Diese können hinzugefügt werden mittels:

```sh
cd ~/src/openstreetmap-carto/
sudo -u _renderd psql -d gis -f functions.sql
```

### Shapefile herunterladen

Obwohl die meisten Daten, die zur Kartenerstellung benötigt werden, direkt aus der zuvor heruntergeladenen Datei mit OpenStreetMap-Daten kommen, sind noch einige Shapefiles nötig für Dinge wie Länder-Grenzen bei niedrigen Zoom-Stufen. Um diese mit dem gleichen Benutzerkonto wie zuvor herunterzuladen und zu indizieren:

```sh
cd ~/src/openstreetmap-carto/
mkdir data
sudo chown _renderd data
sudo -u _renderd scripts/get-external-data.py
```

Dieser Prozess beinhaltet einen beträchtlich Download und könnte einige Zeit dauern - während es läuft, wird auf dem Bildschirm nicht viel erscheinen. Einige Daten werden direkt in die Datenbank gehen und andere werden in einem “data” Verzeichnis unterhalb von “openstreetmap-carto” landen. Sollte hier ein Problem auftreten, dann könnten die Natural Earth Daten umgezogen sein - für mehr Details siehe [dieses Issue](https://github.com/nvkelso/natural-earth-vector/issues/581#issuecomment-913988101){: target=_blank} und andere Issues bei Natural Earth. Wenn Sie die Natural Earth Download-Adresse ändern müssen, dann ist Ihre Kopie [dieser Datei](https://github.com/gravitystorm/openstreetmap-carto/blob/master/external-data.yml){: target=_blank} der richtige Ort zum Editieren.

### Schriftarten

Schriftarten müssen auf diese Weise manuell installiert werden:

```sh
cd ~/src/openstreetmap-carto/
scripts/get-fonts.sh
```

Aufgrund eines [aktuellen Problems](https://github.com/gravitystorm/openstreetmap-carto/issues/5013) erlhalten Sie möglicherweise ganz am Ende davon einen Fehler, es wird sich aber nicht nachteilig auf die Darstellung aktueller OSM-Daten auswirken.

Unsere Test-Daten-Region (Aserbaidschan) wurde gewählt, weil es ein kleines Gebiet ist, und auch, weil einige Ortsbezeichnungen in dieser Region Namen mit nicht-lateinischen Schriftzeichen besitzen.

## Ihren Webserver einrichten

### renderd konfigurieren

Die Konfigurations-Datei für `renderd` auf {{ dist }} ist `/etc/renderd.conf`. Editieren Sie diese mit einem Texteditor wie beispielsweise nano:

```sh
sudo nano /etc/renderd.conf
```

Fügen Sie am Ende einen Abschnitt wie den folgenden hinzu:

```ini
[s2o]
URI=/hot/
TILEDIR=/var/lib/mod_tile
XML=/home/accountname/src/openstreetmap-carto/mapnik.xml
HOST=localhost
TILESIZE=256
MAXZOOM=20
```

Der Speicherort der XML Datei `/home/accountname/src/openstreetmap-carto/mapnik.xml` wird an den tatsächlichen Speicherort auf Ihrem System angepasst werden müssen. Sie können `[s2o]` und `URI=/hot/` ebenfalls anpassen, wenn Sie möchten. Falls Sie mehr als einen Satz an Tiles von einem Server render möchten, können Sie einfach einen weiteren Abschnitt wie `[s2o]` ergänzen, mit einem anderen Namen, der zu einem anderen Karten-Stil verweist. Wenn Sie wollen, dass es auf eine andere Datenbank als das standardmäßige `gis` veweist, dann können Sie das tun, aber das ist nicht mehr im Fokus dieses Dokuments. Wenn Sie nur ungefähr 4Gb Speicher zur Verfügung haben, dann werden Sie auch `num_threads` auf 2 reduzieren wollen. `URI=/hot/` wurde gewählt, so dass die hier erzeugten Tiles einfacher an der Stelle der HOT Tile-Ebene auf OpenStreetMap.org verwendet werden können. Sie können hier etwas beliebiges anderes wählen, aber `/hot/` ist genauso gut wie alles andere.

Der Ort des Mapnik Plugin-Verzeichnis variirt jetzt zwischen verschiedenen Architekturen. Sie können `` 

```sh
sudo updatedb
locate mapnik/4.0/input
```

eingeben, um den Speicherort auf Ihrem Server herauszufinden und dann `/etc/renderd.conf` so editieren, dass `plugins_dir` im `[mapnik]` Abschnitt korrekt ist. Möglicherweise etwa so:


```ini
[mapnik]
plugins_dir=/usr/lib/aarch64-linux-gnu/mapnik/4.0/input
```

Wenn dies falsch gesetzt ist oder ganz fehlt, dann wird unter den Fehlern, die Sie beim Versuch die Tiles zu rendern erhalten, folgendens sein:

!!! warning ""
    An error occurred while loading the map layer 's2o': Could not create datasource for type: 'postgis' (no datasource plugin directories have been successfully registered) encountered during parsing of layer 'landcover-low-zoom'

Andere Fehler könnten under anderem sein, dass Tiles nicht korrekt gerendert werden und dass `renderd` unerwartet große Mengen Speicher verwendet.

etzt, da wir `renderd` gesagt haben, wie auf Tile-Rendering-Anfragen reagiert werden soll, müssen wir dem Apache-Webserver beibringen sie zu senden. Leider wurde die Konfiguration dafür aus der letzten Version von `mod_tile` entfernt. Sie kann jedoch derzeit von hier installiert werden

```sh
cd /etc/apache2/conf-available/
sudo wget https://raw.githubusercontent.com/openstreetmap/mod_tile/python-implementation/etc/apache2/renderd.conf
sudo a2enconf renderd
sudo systemctl reload apache2
```

### Sicherstellen, dass Sie Debug-Nachrichten sehen können

An diesem Punkt wäre es wirklich nützlich die Ausgaben des Tile-Rendering-Prozesses sehen zu können, einschließlich aller Fehlermeldungen. Bei aktuellen `mod_tile` Versionen ist dies standardmäßig deaktiviert. Um es zu aktivieren:

```sh
sudo nano /usr/lib/systemd/system/renderd.service
```

Wenn es unterhalb von `[Service]` nicht bereits vorhanden ist, fügen Sie hinzu:

```ini
Environment=G_MESSAGES_DEBUG=all
```

Dann führen Sie diese Kommandos aus, um die Konfiguration neu zu laden:

```sh
sudo systemctl daemon-reload
sudo systemctl restart renderd
sudo systemctl restart apache2
```

### Apache konfigurieren

```sh
sudo mkdir /var/lib/mod_tile
sudo chown _renderd /var/lib/mod_tile
```

Nachdem Sie dies getan haben, starten Sie `renderd` neu:

```sh
sudo /etc/init.d/renderd restart
```

Wenn Sie sich das System-Log ansehen, sollten Sie Benachrichtigungen vom `renderd` Dienst sehen. Beachten Sie, dass bei den letzten paar Debian Versionen Benachrichtigungen standardmäßig nicht mehr in `/var/log/syslog` geschrieben werden - um Benachrichtigungen zu verfolgen, während sie geschrieben werden, machen Sie dies:

```sh
sudo journalctl -ef
```

Siehe [hier](https://www.debian.org/releases/bookworm/amd64/release-notes/ch-information.en.html#changes-to-system-logging){: target=_blank}: für mehr Details.

```sh
sudo /etc/init.d/apache2 restart
```

Mit `#!sh sudo journalctl -ef` sollten Sie eine Zeile sehen wie:

```log
 Aug 15 12:16:08 h19 apachectl[14590]: [Fri Aug 15 12:16:08.655816 2025] [tile:notice] [pid 14590:tid 14590] Loading tile config s2o at /hot/ for zooms 0 - 20 from tile directory /var/lib/mod_tile with extension .png and mime type image/png
```

Falls dies nicht erscheint, bedeutet es wahrscheinlich, dass die Konfigurations-Datei für mod_tile `/etc/apache2/conf-available/renderd.conf` entweder nicht ordnungsgemäß eingerichtet oder aktiviert ist - sehen Sie sich den `wget` Abschnitt oben an.

Als nächstes besuchen Sie mit einem Webbrowser `http://yourserveripaddress/index.html` (ändern Sie `yourserveripaddress` zu Ihrer tatsächlichen Server-Adresse). Sie sollten "Apache2 Debian Default Page" sehen.

!!! tip
    Falls Sie nicht wissen, welcher IP-Adresse es zugewiesen wurde, dann können Sie voraussichtlich `ip address` benutzen, um es herauszufinden – wenn die Netzwerkkonfiguration nicht zu kompliziert ist, dann ist es wahrscheinlich die `inet addr`, die nicht `127.0.0.1` ist.

Wenn Sie einen Server bei einem Hosting-Anbieter verwenden, dann ist es wahrscheinlich, dass die interne Adresse Ihres Servers sich von der externen Adresse unterscheidet, die Ihnen zugewiesen wurde. Aber diese externe Adresse wird Ihnen bereits übermittelt worden sein und wird vermutlich die sein, über die Sie bereits auf den Server zugreifen.

Beachten Sie, dass dies nur die `http`-Seite (Port 80) ist – Sie werden etwas mehr Apache Konfiguration betreiben müssen, wenn sie `https` aktivieren wollen, aber das ist nicht mehr Teil dieser Anleitung. Wenn Sie jedoch "Let's Encrypt" zur Ausstellung von Zertifikaten benutzen, dann kann der Prozess dies einzurichten auch die Konfiguration der Apache HTTPS-Seite beinhalten.

Als nächstes besuchen Sie mit einem Webbrowser: `http://yourserveripaddress/hot/0/0/0.png`

Falls Sie oben `URI=/hot/` geändert haben, müssen Sie dies hier natürlich ebenfalls anpassen. Sie sollten eine kleine Karte der Welt sehen. Falls nicht, untersuchen Sie die angezeigten Fehlermeldungen. Dies werden höchstwahrscheinlich Berechtigungsfehler sein oder vielleicht damit im Zusammenhang stehen, dass versehentlich Schritten aus der obigen Anleitung ausgelassenen wurden. Wenn Sie kein Tile erhalten und andere Fehler bekommen,speichern Sie die komplette Ausgabe in einen Pastebin und stellen eine Fragen zu den Problem an einem Ort wie [`community.openstreetmap.org`](https://community.openstreetmap.org){: target=_blank}.

## Tiles anzeigen

Um Tiles zu sehen werden wir schummeln und eine html-Datei `sample_leaflet.html` verwenden, die es Ihnen erlaubt eine sehr einfache Karte anzuzeigen. Um sie zu erhalten:

```sh
cd /var/www/html
sudo wget https://raw.githubusercontent.com/SomeoneElseOSM/mod_tile/switch2osm/extra/sample_leaflet.html
sudo nano sample_leaflet.html
```

Passen Sie es so an, dass die IP Adresse `your.server.address` entspricht, statt einfach nur `127.0.0.1`. Damit sollte es Ihnen möglich sein, diesen Server von anderen zu erreichen. Dann navigieren Sie zu `http://your.server.address/sample_leaflet.html`.

Die erstmalige Kartendarstellung wird einen kleinen Moment dauern. Sie werden rein- und rauszoomen können, aber abhängig von der Server-Geschwindigkeit werden einige Tiles zuerst grau dargestellt, weil sie für den Browser nicht rechtzeitig gerendert werden können. obald sie jedoch fertig sind, werden sie für das nächsten mal, wenn sie benötigt werden, bereit sein. Wenn Sie in das System-Log schauen, sollten Sie Anfragen für Tiles sehen.

enn gewünscht, können Sie die Einstellung `ModTileMissingRequestTimeout` in `/etc/apache2/conf-available/renderd.conf` von 10 Sekunden auf 60 oder vielleicht sogar mehr erhöhen, um länger zu warten, bis im Hintergrund Tiles gerendert werden (wenn es keine alten gibt) bevor eine graue Tile an den Benutzer gegeben wird. Stellen Sie sicher, dass Sie `#!sh sudo service renderd restart` und `#!sh sudo service apache2 restart` ausführen, nachdem Sie es geändert haben. Vielleicht möchten Sie sogar die Auswirkung von Änderungen an anderen Variablen dort messen.

Glückwunsch! Schauen Sie in die [Tiles verwenden](/using-tiles/index.md)-Bereich um eine Karte zu erstellen, die Ihren neuen Tile-Server verwendet.
