# AzerothCore WotLK Server Setup

Diese Anleitung dokumentiert den reproduzierbaren Aufbau dieses Repositories als privaten **World of Warcraft 3.3.5a (Build 12340)** Server mit **AzerothCore**, **mod-playerbots** und **mod-ah-bot**.

> **Wichtig:** Das Repository enthält absichtlich keine `.env`, keine Passwörter, keine Client-Daten und keine Datenbank-Backups. Ein Git-Clone allein stellt daher Quellcode und Konfiguration wieder her, aber nicht Accounts, Charaktere oder den bestehenden Weltzustand.

## 1. Was dieses Repository enthält

Der Default-Branch ist `Playerbot`. Die beiden zusätzlichen Module sind als Git-Submodules eingebunden:

- `modules/mod-playerbots` → `https://github.com/mod-playerbots/mod-playerbots.git`, Branch `master`
- `modules/mod-ah-bot` → `https://github.com/azerothcore/mod-ah-bot.git`, Branch `master`

Die im Repository gepinnten Revisionen sind zum Zeitpunkt dieser Dokumentation:

```text
mod-playerbots  b6696bdbd3740e575598d167d69f39f68cc0b907
mod-ah-bot      a680cc1c98290713e9b3d3289544af78e5186dc1
```

Wichtige Dateien:

```text
docker-compose.yml
	das vom Playerbot-Fork gelieferte Basis-Compose-Setup

docker-compose.override.yml
	unsere Server-Anpassungen; Basisdatei nicht direkt verändern

conf/dist/modules/playerbots.conf
	Playerbots-Konfiguration

conf/dist/modules/mod_ahbot.conf
	Auction-House-Bot-Konfiguration

.gitmodules
	Definition der beiden Module

conf/dist/env.docker
	Vorlage/Referenz für lokale .env-Variablen
```

## 2. Voraussetzungen

Empfohlen ist ein aktuelles Debian-System mit x86_64/amd64. Das getestete Deployment lief auf Debian 13 mit 8 vCPU und 15 GiB RAM.

Benötigt werden mindestens:

```text
Git
Docker Engine
Docker Compose Plugin (docker compose)
```

Für etwa 1000 gleichzeitig aktive Random Bots sollte der Host mehrere CPU-Kerne und ausreichend RAM besitzen. Andere Gameserver auf demselben Host müssen bei der Kapazitätsplanung berücksichtigt werden.

Extern erreichbar müssen für WoW normalerweise nur diese TCP-Ports sein:

```text
3724/tcp   Authserver
8085/tcp   Worldserver
```

MySQL (`3306`) und SOAP (`7878`) sollten nicht öffentlich erreichbar sein, sofern kein konkreter Grund dafür besteht.

## 3. Repository klonen

Beispiel für den auf dem bestehenden Server verwendeten Zielpfad:

```bash
mkdir -p /media/data/src
cd /media/data/src

git clone \
  --branch Playerbot \
  --recurse-submodules \
  https://github.com/PragmaticBeaver/azerothcore-wotlk.git

cd azerothcore-wotlk
```

Danach prüfen:

```bash
git status
git submodule status
```

Falls das Repository ohne Submodules geklont wurde:

```bash
git submodule update --init --recursive
```

Die Module dürfen beim Build nicht fehlen. Ein Worldserver, der ohne `mod-playerbots` gebaut wurde, kann zwar starten, besitzt aber keine Playerbot-Funktionalität.

## 4. `.env` erstellen

Die `.env` wird **nicht committed**. Als Ausgangspunkt dient:

```bash
cp conf/dist/env.docker .env
```

Für ein Linux-Deployment sollte sie mindestens sinnvoll gesetzte Werte für Benutzer, Ports und Datenbankpasswort enthalten. Beispiel:

```dotenv
DOCKER_USER=YOUR_LINUX_USER
DOCKER_USER_ID=1000
DOCKER_GROUP_ID=1000

DOCKER_AUTH_EXTERNAL_PORT=3724
DOCKER_WORLD_EXTERNAL_PORT=8085

# Nur lokal veröffentlichen:
DOCKER_DB_EXTERNAL_PORT=127.0.0.1:3306
DOCKER_SOAP_EXTERNAL_PORT=127.0.0.1:7878

# Unbedingt durch ein starkes, einzigartiges Passwort ersetzen:
DOCKER_DB_ROOT_PASSWORD=CHANGE_ME
```

Die Basis-Compose-Datei besitzt für `DOCKER_DB_ROOT_PASSWORD` den Fallback `password`. Auf einem echten Server darf man sich **nicht** auf diesen Default verlassen.

Die Form `127.0.0.1:3306` ist beabsichtigt: `docker-compose.yml` ergänzt daran `:3306` für den Containerport, sodass MySQL nur an Loopback gebunden wird. Dasselbe Prinzip gilt für SOAP.

Effektive Compose-Konfiguration prüfen:

```bash
docker compose config
```

Besonders auf veröffentlichte Ports achten:

```bash
docker compose config | grep -E '3306|3724|7878|8085'
```

## 5. Verzeichnisse und Rechte

Die Container werden mit der in `.env` angegebenen UID/GID gebaut. Die beschreibbaren Host-Verzeichnisse müssen dazu passen:

```bash
mkdir -p env/dist/etc env/dist/logs
sudo chown -R 1000:1000 env/dist/etc env/dist/logs
```

Die Modul-Konfigurationen liegen im Repository und werden vom Override read-only in den Worldserver gemountet.

Falls bei `ac-db-import` Meldungen wie `Permission denied` für `/azerothcore/env/dist/etc` oder `/azerothcore/env/dist/logs` erscheinen, zuerst Eigentümer und die Werte `DOCKER_USER_ID` / `DOCKER_GROUP_ID` prüfen.

## 6. Client-Daten

AzerothCore benötigt die extrahierten Client-Daten:

```text
dbc/
maps/
mmaps/
vmaps/
Cameras/
```

Das Compose-Setup verwendet dafür standardmäßig das Docker-Volume:

```text
ac-client-data
```

Beim aktuellen Projekt heißt das von Compose erzeugte Volume typischerweise:

```text
azerothcore-wotlk_ac-client-data
```

### Variante A: Compose-Client-Data-Image

Das Repository besitzt den Service `ac-client-data-init`. Bei einem normalen Compose-Start wird dieser als Dependency des Worldservers ausgeführt.

### Variante B: bestehende Client-Daten wiederherstellen

Für eine Migration wurde das Volume erfolgreich als Tarball gesichert und auf dem Zielserver wiederhergestellt. Ein Backup kann z. B. so erstellt werden:

```bash
docker run --rm \
  -v azerothcore-wotlk_ac-client-data:/data:ro \
  -v "$PWD":/backup \
  alpine \
  tar czf /backup/ac-client-data.tar.gz -C /data .
```

Auf dem Zielserver:

```bash
docker volume create azerothcore-wotlk_ac-client-data

docker run --rm \
  -v azerothcore-wotlk_ac-client-data:/data \
  -v "$PWD":/backup \
  alpine \
  sh -c 'rm -rf /data/* && tar xzf /backup/ac-client-data.tar.gz -C /data'
```

Prüfen:

```bash
docker run --rm \
  -v azerothcore-wotlk_ac-client-data:/data:ro \
  alpine ls -lah /data
```

**Client-Daten niemals ins Git-Repository committen.**

## 7. Datenbank initialisieren

Die Basis-Konfiguration verwendet MySQL 8.4 und folgende Datenbanken:

```text
acore_auth
acore_characters
acore_world
acore_playerbots
```

Zuerst die Datenbank starten:

```bash
docker compose up -d ac-database
```

Status prüfen:

```bash
docker compose ps
```

Anschließend den Import-Service ausführen:

```bash
docker compose up ac-db-import
```

Das Override ergänzt für diesen Service die Playerbot-Datenbankverbindung:

```text
AC_PLAYERBOTS_DATABASE_INFO
```

Danach prüfen:

```bash
docker compose exec -T ac-database \
  mysql -uroot -p"$DOCKER_DB_ROOT_PASSWORD" \
  -e 'SHOW DATABASES;'
```

Falls die Shellvariable nicht exportiert ist, das Passwort aus `.env` verwenden oder die Variable vorher sicher laden. Passwörter nicht in Shell-History, Dokumentation oder Git eintragen.

## 8. mod-ah-bot SQL importieren

`mod-ah-bot` verlangt laut Modul-Dokumentation einen manuellen Import seiner World-DB-SQL-Dateien. In der gepinnten Modulversion liegen sie hier:

```text
modules/mod-ah-bot/data/sql/db-world/mod_auctionhousebot.sql
modules/mod-ah-bot/data/sql/db-world/auctionhousebot_professionItems.sql
modules/mod-ah-bot/data/sql/db-world/z_filter_disabled_and_trash.sql
```

In dieser Reihenfolge importieren:

```bash
docker compose exec -T ac-database \
  mysql -uroot -p"$DOCKER_DB_ROOT_PASSWORD" acore_world \
  < modules/mod-ah-bot/data/sql/db-world/mod_auctionhousebot.sql

docker compose exec -T ac-database \
  mysql -uroot -p"$DOCKER_DB_ROOT_PASSWORD" acore_world \
  < modules/mod-ah-bot/data/sql/db-world/auctionhousebot_professionItems.sql

docker compose exec -T ac-database \
  mysql -uroot -p"$DOCKER_DB_ROOT_PASSWORD" acore_world \
  < modules/mod-ah-bot/data/sql/db-world/z_filter_disabled_and_trash.sql
```

Die SQL-Dateien legen standardmäßig Konfigurationen für AH IDs `2` (Alliance), `6` (Horde) und `7` (Neutral) an.

## 9. Server aus dem Repository bauen

Da Playerbots und AHBot C++-Module sind, muss der Worldserver aus diesem Repository gebaut werden. Nicht einfach dauerhaft ein fremdes vorgebautes `master`-Image verwenden.

Sauberer Erstbuild:

```bash
docker compose build --no-cache ac-worldserver ac-authserver ac-db-import
```

Danach:

```bash
docker compose up -d
```

Status:

```bash
docker compose ps
```

Logs:

```bash
docker compose logs -f ac-worldserver
```

Der Worldserver sollte als `Playerbot branch` erkennbar sein und Playerbots initialisieren.

Gezielte Kontrolle:

```bash
docker compose logs ac-worldserver \
  | grep -iE 'playerbot|random bot|auction|ahbot'
```

## 10. Aktuelle Server-Tuning-Werte

`docker-compose.override.yml` setzt aktuell effektiv:

```text
Random Bots                 1000 min / 1000 max
Playerbots                  aktiviert
RandomBot Autologin         aktiviert
Player Limit                0
Map Update Threads          4

XP Kill                     2x
XP Quest                    2x
XP Explore                  2x
XP Pet                      2x

Drop Poor                   2x
Drop Normal                 2x
Drop Uncommon               2x
Drop Rare                   1.5x
Drop Epic                   1x
```

Wichtig: `conf/dist/modules/playerbots.conf` enthält derzeit noch:

```text
AiPlayerbot.MinRandomBots = 1500
AiPlayerbot.MaxRandomBots = 1500
```

Das ist **kein Widerspruch im laufenden Docker-Setup**: Die Environment-Variablen im Compose-Override überschreiben diese Werte effektiv auf 1000. Für Diagnose immer zusätzlich `docker compose config` prüfen.

Weitere bewusst gesetzte Playerbot-Werte in der Config sind unter anderem:

```text
AiPlayerbot.RandomBotAccountCount = 200
AiPlayerbot.AddClassAccountPoolSize = 50
AiPlayerbot.BotAutologin = 0
AiPlayerbot.AllowAccountBots = 1
AiPlayerbot.AllowGuildBots = 1
AiPlayerbot.AllowTrustedAccountBots = 1
AiPlayerbot.AutoEquipUpgradeLoot = 1
```

Damit können insbesondere separate Accounts als persönliche Altbot-/Companion-Accounts verknüpft werden.

## 11. AHBot-Charakter einrichten

Die Repository-Konfiguration enthält derzeit:

```text
AuctionHouseBot.EnableSeller = 1
AuctionHouseBot.EnableBuyer = 1
AuctionHouseBot.Account = 203
AuctionHouseBot.GUID = 2002
AuctionHouseBot.ConsiderOnlyBotAuctions = 1
AuctionHouseBot.DuplicatesCount = 5
AuctionHouseBot.DivisibleStacks = 1
AuctionHouseBot.ProfessionItems = 1
```

**Account `203` und GUID `2002` sind installationsspezifisch und nicht portabel.** Bei einer frischen Datenbank müssen ein AHBot-Account und ein Charakter angelegt und anschließend die tatsächlichen IDs in `conf/dist/modules/mod_ahbot.conf` eingetragen werden.

Account über die Worldserver-Konsole erstellen:

```bash
docker attach ac-worldserver
```

Dann beispielsweise:

```text
account create ahbot EIN_SICHERES_PASSWORT
```

Mit `Ctrl+P`, danach `Ctrl+Q` vom Container lösen, ohne ihn zu stoppen.

Mit dem neuen Account einmal im WoW-Client einloggen und einen dedizierten AHBot-Charakter erstellen. Dieser Charakter ist nicht zum normalen Spielen gedacht.

IDs danach auslesen:

```sql
SELECT id, username FROM acore_auth.account WHERE username = 'AHBOT';
SELECT guid, account, name FROM acore_characters.characters WHERE account = <ACCOUNT_ID>;
```

Über Docker z. B.:

```bash
docker compose exec -T ac-database \
  mysql -uroot -p"$DOCKER_DB_ROOT_PASSWORD" \
  -e "SELECT id,username FROM acore_auth.account WHERE username='AHBOT';"
```

Die gefundenen Werte in `conf/dist/modules/mod_ahbot.conf` eintragen und Worldserver neu erstellen/starten.

### AH-Mengen

Die Mengen liegen in `acore_world.mod_auctionhousebot`, nicht in `mod_ahbot.conf`. Die IDs sind:

```text
2 = Alliance
6 = Horde
7 = Neutral
```

Im bisherigen produktiven Datenbankzustand wurden als Zielwerte verwendet:

```text
Alliance  2000–3000
Horde     2000–3000
Neutral    500–800
```

Diese Werte sind **Datenbankzustand und nicht vollständig durch dieses Git-Repository reproduziert**. Bei einer frischen DB müssen sie erneut gesetzt oder aus einem DB-Backup übernommen werden. Vor Änderungen zuerst die aktuelle Tabellenstruktur prüfen:

```sql
DESCRIBE acore_world.mod_auctionhousebot;
SELECT auctionhouse,name,minitems,maxitems FROM acore_world.mod_auctionhousebot;
```

Dann z. B.:

```sql
UPDATE acore_world.mod_auctionhousebot
SET minitems=2000, maxitems=3000
WHERE auctionhouse IN (2,6);

UPDATE acore_world.mod_auctionhousebot
SET minitems=500, maxitems=800
WHERE auctionhouse=7;
```

## 12. Realm-Adresse konfigurieren

Bei einer frischen Datenbank die Realm-Adresse auf die vom Client erreichbare Adresse setzen:

```sql
SELECT id,name,address,localAddress,localSubnetMask,port
FROM acore_auth.realmlist;
```

Für einen öffentlichen Server beispielsweise:

```sql
UPDATE acore_auth.realmlist
SET address='<SERVER_IP_OR_HOSTNAME>',
    localAddress='<SERVER_IP_OR_HOSTNAME>',
    port=8085
WHERE id=1;
```

Die korrekte Wahl von `localAddress` hängt vom Netzwerkaufbau ab. Bei NAT/LAN-Setups nicht blind eine öffentliche Adresse übernehmen.

Im WoW-3.3.5a-Client in `realmlist.wtf`:

```text
set realmlist <SERVER_IP_OR_HOSTNAME>
```

## 13. Ersten Spieleraccount erstellen

Worldserver-Konsole öffnen:

```bash
docker attach ac-worldserver
```

Account anlegen:

```text
account create ACCOUNTNAME PASSWORT
```

Optional GM-Level 3 vergeben:

```text
account set gmlevel ACCOUNTNAME 3 -1
```

Die genaue Command-Syntax kann sich mit AzerothCore-Versionen ändern; bei Unsicherheit zuerst in der Worldserver-Konsole `help account` bzw. `help account set gmlevel` verwenden.

## 14. Persönlichen Playerbot-Companion erstellen

Das aktuelle `playerbots.conf` erlaubt Trusted Accounts:

```text
AiPlayerbot.AllowTrustedAccountBots = 1
```

Bot-Account erstellen:

```text
account create BOTACCOUNT PASSWORT
```

Danach mit diesem Account im Client einen normalen Charakter erstellen.

Mit dem Bot-Charakter einloggen:

```text
.playerbots account setKey TEMPORAERER_KEY
```

Mit dem Spieleraccount einloggen:

```text
.playerbots account link BOTACCOUNT TEMPORAERER_KEY
.playerbots account linkedAccounts
```

Companion verwenden:

```text
.playerbots bot add BOTNAME
.playerbots bot remove BOTNAME
```

Die Verknüpfung ist **Account ↔ Account**, nicht Charakter ↔ Charakter. Dadurch kann derselbe Spieleraccount unterschiedliche Solo-/Coop-Charaktere haben und den Companion nur dort hinzufügen, wo er gewünscht ist.

Das aktuell verwendete Beispiel ist:

```text
Bot-Account: domebot
Bot-Charakter: Leonard
```

Keine Passwörter oder Trusted-Account-Keys dokumentieren.

## 15. Random Bots prüfen

Effektive Compose-Werte:

```bash
docker compose config | grep -E 'AC_AI_PLAYERBOT_(MIN|MAX)_RANDOM_BOTS'
```

Online-Charaktere zählen:

```bash
docker compose exec -T ac-database \
  mysql -uroot -p"$DOCKER_DB_ROOT_PASSWORD" acore_characters \
  -e 'SELECT COUNT(*) AS online_characters FROM characters WHERE online=1;'
```

Playerbot-DB prüfen:

```bash
docker compose exec -T ac-database \
  mysql -uroot -p"$DOCKER_DB_ROOT_PASSWORD" acore_playerbots \
  -e 'SHOW TABLES;'
```

Die in dieser Modulrevision verwendete Random-Bot-Tabelle heißt `playerbots_random_bots`.

## 16. Container-Mounts prüfen

Ein häufiger Fehler beim ersten Deployment war ein fehlendes `docker-compose.override.yml`. Dann wurde zwar der Playerbot-Fork gebaut, aber Module und Modul-Konfigurationen fehlten im laufenden Container.

Prüfen:

```bash
docker inspect ac-worldserver \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Erwartet werden zusätzlich zu `etc`, `logs` und Client-Daten insbesondere Mounts für:

```text
./modules -> /azerothcore/modules
playerbots.conf -> /azerothcore/env/dist/etc/modules/playerbots.conf
mod_ahbot.conf -> /azerothcore/env/dist/etc/modules/mod_ahbot.conf
```

Nach Mount- oder Compose-Änderungen reicht `docker restart` nicht zuverlässig. Container neu erstellen:

```bash
docker compose up -d --force-recreate ac-worldserver
```

Nach Änderungen am C++-Code oder an Submodules zusätzlich neu bauen:

```bash
docker compose build --no-cache ac-worldserver
docker compose up -d --force-recreate ac-worldserver
```

## 17. Datenbank-Backup und Restore

Für eine vollständige Migration wurden alle vier relevanten Datenbanken gemeinsam gesichert:

```bash
docker compose exec -T ac-database mysqldump \
  -uroot -p"$DOCKER_DB_ROOT_PASSWORD" \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --databases \
  acore_auth acore_characters acore_playerbots acore_world \
  > azerothcore-full.sql
```

Restore auf einer laufenden, leeren MySQL-Instanz:

```bash
docker compose exec -T ac-database \
  mysql -uroot -p"$DOCKER_DB_ROOT_PASSWORD" \
  < azerothcore-full.sql
```

Zusammen mit dem Backup des `ac-client-data`-Volumes lässt sich damit der komplette Spielzustand auf einen neuen Host migrieren.

**`azerothcore-full.sql` niemals committen.** Es enthält Account- und Charakterdaten.

## 18. Updates

Vor einem Update zuerst DB-Backup erstellen.

Dann:

```bash
git pull origin Playerbot
git submodule update --init --recursive
```

Submodule nicht ungeprüft auf den neuesten Stand ziehen. Das Repository pinnt bewusst getestete Commits. Wenn Module aktualisiert werden, Änderungen explizit committen und danach sauber neu bauen.

Nach Core-/Moduländerungen:

```bash
docker compose build --no-cache ac-worldserver ac-authserver ac-db-import
docker compose up -d --force-recreate
```

Danach Logs und DB-Updater-Ausgaben kontrollieren.

## 19. Sicherheitscheck

Nach einem frischen Deployment mindestens prüfen:

```bash
ss -lntp

docker compose ps
```

Von einem externen Rechner sollten `3724` und `8085` erreichbar sein. `3306` und `7878` sollten bei der empfohlenen `.env` **nicht** öffentlich erreichbar sein.

Weitere Regeln:

- starkes, einzigartiges `DOCKER_DB_ROOT_PASSWORD`
- `.env` niemals committen
- SQL-Backups niemals committen
- keine WoW-Client-Dateien/Map-Daten committen
- nur tatsächlich benötigte Ports veröffentlichen
- vor Updates Backups erstellen

## 20. Bekannte Stolperfallen aus dem ersten Aufbau

### Worldserver startet, aber keine Playerbot-Kommandos

Prüfen, ob wirklich der eigene `Playerbot`-Branch gebaut wurde und die Submodules vorhanden sind:

```bash
git branch --show-current
git submodule status
docker compose logs ac-worldserver | head -100
```

### `mod-playerbots` fehlt im Container

`docker-compose.override.yml` bzw. die Mounts prüfen und den Container **recreaten**.

### `ac-db-import` meldet Permission denied

UID/GID aus `.env` und Eigentümer von `env/dist/etc` / `env/dist/logs` prüfen.

### MySQL ist versehentlich öffentlich

In `.env`:

```dotenv
DOCKER_DB_EXTERNAL_PORT=127.0.0.1:3306
```

Dann DB-Container recreaten und mit `ss` sowie von extern testen.

### Host-Config geändert, Container sieht alten Inhalt

Bei bind-mounted Einzeldateien können Datei-Ersetzungen (`sed -i`, Editor-Save via Rename) dazu führen, dass ein laufender Container noch den alten Inode gemountet hat. In diesem Fall Worldserver recreaten:

```bash
docker compose up -d --force-recreate ac-worldserver
```

### `playerbots.conf` sagt 1500, aber Server nutzt 1000

Das Compose-Override setzt `AC_AI_PLAYERBOT_MIN_RANDOM_BOTS` und `AC_AI_PLAYERBOT_MAX_RANDOM_BOTS` auf 1000 und hat damit im Docker-Setup Vorrang. Immer die effektive Compose-Konfiguration prüfen.

## 21. Reproduzierbarkeit: Was Git kann und was nicht

Mit diesem Repository reproduzierbar:

```text
AzerothCore/Playerbot-Quellcode
gepinntes mod-playerbots
gepinntes mod-ah-bot
Playerbot-Konfiguration
AHBot-Konfiguration
Docker-Override
XP-/Drop-Rates
Build-Struktur
```

Nicht allein aus Git reproduzierbar:

```text
.env und Secrets
Accounts und Charaktere
AHBot Account/GUID einer frischen Installation
Realm-IP/Hostname
AHBot min/max-Werte, sofern nur in der DB geändert
bestehende Auktionen
Random-Bot-Datenbankzustand
Spielerfortschritt
Client-Daten (dbc/maps/mmaps/vmaps/Cameras)
```

Für eine **exakte Wiederherstellung** deshalb immer drei Dinge sichern:

1. dieses Git-Repository bzw. den Remote-Stand,
2. einen Dump der vier AzerothCore-Datenbanken,
3. das `ac-client-data`-Volume.

Damit ist der Server sowohl frisch reproduzierbar als auch vollständig migrierbar.
