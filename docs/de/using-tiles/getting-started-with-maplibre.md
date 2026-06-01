---
layout: docs
title: Erste Schritte mit MapLibre GL
---

# {{ title }}

## Einleitung

[MapLibre GL JS](https://maplibre.org/maplibre-gl-js/docs/){: target=blank} ist eine TypeScript-Library, die WebGL zur Einbindung von Karten nutzt. Es verwendet eine freizügige BSD Open-Source Lizenz - kann also ohne Bedenken legal in jede Seite integriert werden. Der Quellcode ist auf [GitHub](https://github.com/maplibre/maplibre-gl-js/){: target=blank} verfügbar.

Wir beschränken uns hier auf ein kleines, eigenständiges Beispiel und verweisen für komplexere Anwendungen auf die offiziellen [Tutorials](https://maplibre.org/maplibre-gl-js/docs/examples/){: target=blank} und die [Dokumentation](https://maplibre.org/maplibre-gl-js/docs/API/){: target=blank}.

## Erste Schritte

Eine Karte darzustellen erfordert drei Dinge: Eine Datenquelle, ein Karten-Stil und eine Seite um alles zu hosten. Wir werden die Shortbread-Tiles der OpenStreetMap Stiftung verwenden, den Versatiles Colorful Stil und eine Webseite, die Sie betreiben.

!!! info "Hosting"
    Manche Browser-Funktionen erwarten, dass eine Seite von einem sicheren Speicherort ausgeliefert wird. Dies kann eine Webseite mit HTTPS oder Ihr lokaler Computer sein. Wir nehmen an, dass Sie einen Webhoster für "example.com" haben, der es Ihnen erlaubt Dateien von einem Datenträger auszuliefern, und dass Sie wissen wie man Dateien zu diesem Webhoster kopiert.

## Software Installation

Wir benötigen Node.js 18 oder höher. Wenn Sie Ubuntu 24.04 (oder höher) oder Debian 12 (oder höher) verwenden, dann können Sie dies mit folgendem Kommando installieren

```sh
sudo apt-get install nodejs
```

Für andere Betriebssysteme schlagen Sie in deren Dokumentation nach. Sie können ihre Version von Node.js mit `node --version` überprüfen. Wenn es niedriger als 18 ist, müssen Sie ihre eigene Version von Node installieren. Eine Möglichkeit dies zu tun ist mittels [nvm](https://github.com/nvm-sh/nvm).

## Den Stil erstellen

Wir werden den VersaTiles Colorful Style benutzen - ein grundlegender Stil, der Shortbread-Tiles verwendet. Das Shortbread Vector-Tile-Schema ist ein universelles Vector-Tile-Schema für OpenStreetMap Daten. Um die Tiles zu erhalten, werden wir den Shortbread Tile-Dienst der OpenStreetMap Stiftung benutzen.

!!! info ""
    Die Verwendung der Vector-Tiles wird durch die [Vector Tile Usage Policy](https://operations.osmfoundation.org/policies/vector/) geregelt. Diese Webseite wird die Anforderungen erfüllen, aber es gibt keine Dienstleistungsvereinbarung oder Garantie für den Vector-Tile-Dienst. Wenn Sie diese benötigen, dann sollten Sie Ihre eigenen Tiles bereitstellen oder einen kommerziellen Anbieter verwenden.

Ein Stil benötigt eine Stil-Definition und Sprite-Dateien für jedes Icon. Wir werden die Stil-Definition erstellen um zu unseren eigenen Sprite-Dateien zu verweisen.

Starten Sie damit, ein neues Verzeichnis zu erstellen, um die Dateien zu speichern, die Sie erzeugen werden. Wir werden es in der Dokumentation "style" nennen, aber es kann heißen wie Sie wollen. Innerhalb dieses Verzeichnisses werden wir alle benötigten Dateien erstellen und in ein "release" Unterverzeichnis ablegen

```sh
mkdir style
cd style
mkdir release
```

Sprites zu erzeugen kann ein komplizierter Prozess sein, aber da wir sie nicht modifizieren müssen, werden wir vorgefertigte verwenden

```sh
curl -OL https://github.com/versatiles-org/versatiles-style/releases/download/v5.10.0/sprites.tar.gz
mkdir -p release/sprites
tar -C release/sprites -xzf sprites.tar.gz
```

Jetzt müssen wir den Stil erstellen, sodass er unsere neue Kopie der Sprites und den OSM Vektor-Tile-Dienst verwendet.

Kopieren Sie den folgenden Inhalt in eine Datei [build.ts](build.ts){: target=_blank}, aber ändern Sie "example.com" in die URL von der Sie die Tiles anbieten werden, einschließlich Ihres Domain-Namen

```ts title="build.ts"
--8<-- "docs/assets/using-tiles/build.ts"
```

Im gleichen Verzeichnis installieren Sie TypeScript und die VersaTiles-Stile und führen dann obiges Script aus, um Ihre Stile zu erstellen.

```sh
npm install tsx @versatiles/style@~5.10.0
node_modules/.bin/tsx build.ts
```

Kopieren Sie den folgenden Inhalt in eine Datei [maplibre.html](maplibre.html){: target=_blank} und legen Sie sie im release Verzeichnis ab

```html title="maplibre.html"
--8<-- "docs/en/using-tiles/maplibre.html"
```

## Den Stil freigeben

Kopieren Sie den Inhalt des "release" Verzeichnisses an den vorher gewählten Ort, von dem aus Sie die Tiles ausliefern wollen. Übliche Arten dies zu tun sind mit einem scp oder rsync Kommando oder über eine Web-Oberfläche.

## Häufige Probleme

### `node_modules/.bin/tsx build.ts` lässt sich nicht ausführen

Wenn Sie eine veraltete Version von node verwenden, wird dieses Kommando fehlschlagen. Sie können dies beheben, indem Sie, wie oben beschrieben, eine aktuelle Version mit nvm installieren.

### Auf der Webseite wird nichts geladen

Öffnen Sie die Entwickler-Werkzeuge Ihres Browsers und sehen Sie sich die Konsole an. Die häufigste Ursache für fehlende Darstellung, sind falsche URLs.
