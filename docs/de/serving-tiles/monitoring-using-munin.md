---
layout: docs
title: Monitoring mit Hilfe von Munin
lang: de
---

# {{ title }}

"Munin" kann verwendet werden, um die Aktivitäten von "renderd" und "mod_tile" auf einem Server zu monitoren. Munin ist auf einer Reihe von Plattformen verfügbar; diese Anleitung wurde auf Ubuntu Linux 22.04 im Juni 2022 getestet.

Als erstes installieren Sie die notwendige Software:

```sh
sudo apt install munin-node munin libcgi-fast-perl libapache2-mod-fcgid
```

Wenn Sie sich `/etc/apache2/conf-available` ansehen, sollten Sie, dass `munin.conf` ein symbolischer Link zu `../../munin/apache24.conf` ist, was `/etc/munin/apache24.conf` entspricht.

Die Datei `/etc/munin/apache24.conf` ist Apaches `munin` Konfigurations-Datei. Wenn Sie wollen, dass auf `munin` global statt nur lokal zugeriffen wird, dann ändern Sie in dieser Datei beide Vorkommen von `Require local` zu `Require all granted`.

Als nächstes editieren Sie `/etc/munin/munin.conf`. Kommetieren Sie diese Zeilen ein:

```conf
dbdir /var/lib/munin
htmldir /var/cache/munin/www
logdir /var/log/munin
rundir /var/run/munin
```

Starten Sie munin und apache neu:

```sh
sudo /etc/init.d/munin-node restart
sudo /etc/init.d/apache2 restart
```

Navigieren Sie zu `http://yourserveripaddress/munin`. Sie sollten eine Seite sehen, die "apache", "disk", "munin", etc. anzeigt.

Um die Plugins von mod_tile und renderd zu munin hinzuzufügen:

```sh
sudo ln -s /usr/share/munin/plugins/mod_tile* /etc/munin/plugins/
sudo ln -s /usr/share/munin/plugins/renderd* /etc/munin/plugins/
```

Es sollten 4 mod_tile-Plugins und 5 renderd-Plugins sein. Starten Sie munins cron-Job einmalig manuell:

```sh
sudo -u munin munin-cron
```

Starten Sie munin und apache ein weiteres mal neu:

```sh
sudo /etc/init.d/munin-node restart
sudo /etc/init.d/apache2 restart
```

Nach einer kurzen Verzögerung sollte jetzt eine Aktualisierung von `http://yourserveripaddress/munin/` Einträge für "mod_tile" und "renderd" zeigen.

Munin aktualisiert seine Diagramme alle 5 Minuten, so wie es durch die cron-Datei `/etc/cron.d/munin` konfiguriert ist.
