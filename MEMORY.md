# AzerothCore Server Context for ChatGPT

> Diese Datei ist als technischer Kontext für einen zukünftigen ChatGPT-Chat gedacht. Lies sie zuerst, bevor du bei Problemen mit diesem Server Annahmen über Architektur, Module oder Konfiguration triffst.
>
> **Regel für den Assistenten:** Behandle Werte aus dieser Datei als dokumentierten Stand, nicht als ewige Wahrheit. Bei Debugging zuerst den aktuellen Repository-Stand, `docker compose config`, Logs und Datenbankzustand prüfen. Keine Secrets erfragen oder erfinden.

## 1. Ziel des Servers

Privater World-of-Warcraft-Server für **Wrath of the Lich King 3.3.5a, Build 12340** auf AzerothCore.

Ziele:

- primär wenige echte Spieler (typischerweise 1–2)
- Welt soll durch Playerbots belebt wirken
- ungefähr 1000 Random Bots gleichzeitig
- Bots sollen questen, leveln und RPG-Aktivitäten ausführen
- Auction House soll durch AHBot benutzbar und belebt sein
- persönliche Altbot-Companions sollen möglich sein
- Solo-Charaktere können mit Companion spielen
- Coop-Charaktere können ohne Companion gespielt werden
- Server soll reproduzierbar über Git + Docker aufsetzbar sein

LLM/Ollama-Chat für Bots wurde diskutiert, ist **nicht Teil des aktuellen Scopes**.

## 2. Repository

```text
Repository: https://github.com/PragmaticBeaver/azerothcore-wotlk
Default branch: Playerbot
```

Das Repository ist ein Fork des Playerbot-AzerothCore-Forks.

Bekannter lokaler Entwicklungs-Pfad aus dem ursprünglichen Setup:

```text
/home/dome/src/cloud-server-setup/wow-server/azerothcore-wotlk
```

Bekannter Deployment-Pfad auf dem Server:

```text
/media/data/src/azerothcore-wotlk
```

Diese Pfade sind installationsspezifisch und bei einem neuen Host nicht voraussetzen.

## 3. Module

Git-Submodules laut `.gitmodules`:

```text
modules/mod-playerbots
  https://github.com/mod-playerbots/mod-playerbots.git
  branch master

modules/mod-ah-bot
  https://github.com/azerothcore/mod-ah-bot.git
  branch master
```

Zum dokumentierten Zeitpunkt im Repository gepinnt:

```text
mod-playerbots: b6696bdbd3740e575598d167d69f39f68cc0b907
mod-ah-bot:     a680cc1c98290713e9b3d3289544af78e5186dc1
```

Bei Fehleranalysen **nicht automatisch davon ausgehen, dass diese SHAs noch aktuell sind**. Zuerst:

```bash
git submodule status
```

## 4. Docker-Architektur

Relevante Services aus `docker-compose.yml`:

```text
ac-database          MySQL 8.4
ac-db-import         DB-Import/Updater
ac-worldserver       Worldserver
ac-authserver        Authserver
ac-client-data-init  AzerothCore Client-Daten
```

Netzwerk:

```text
ac-network
```

Wichtige Volumes:

```text
ac-database
ac-client-data
```

Mit Compose-Projektnamen kann daraus beispielsweise werden:

```text
azerothcore-wotlk_ac-database
azerothcore-wotlk_ac-client-data
```

## 5. `.env`

`.env` ist absichtlich nicht im Repository.

Vorlage:

```text
conf/dist/env.docker
```

Relevante Variablen:

```text
DOCKER_AC_ENV_FILE
DOCKER_VOL_ETC
DOCKER_VOL_LOGS
DOCKER_VOL_DATA
DOCKER_WORLD_EXTERNAL_PORT
DOCKER_SOAP_EXTERNAL_PORT
DOCKER_AUTH_EXTERNAL_PORT
DOCKER_DB_EXTERNAL_PORT
DOCKER_DB_ROOT_PASSWORD
DOCKER_USER
DOCKER_USER_ID
DOCKER_GROUP_ID
```

Sicherheitsziel:

```dotenv
DOCKER_DB_EXTERNAL_PORT=127.0.0.1:3306
DOCKER_SOAP_EXTERNAL_PORT=127.0.0.1:7878
```

MySQL und SOAP sollen nicht öffentlich erreichbar sein, sofern nicht bewusst anders konfiguriert.

**Niemals den Default `password` als produktives MySQL-Root-Passwort empfehlen.**

## 6. Öffentliche WoW-Ports

```text
3724/tcp  Authserver
8085/tcp  Worldserver
```

Historisch verwendete öffentliche Server-IP:

```text
159.195.198.128
```

Diese IP ist nur historischer Kontext. Bei einem zukünftigen Problem die aktuelle IP/Domain prüfen, statt sie blind zu verwenden.

## 7. Aktuelles Compose-Override

Datei:

```text
docker-compose.override.yml
```

Dokumentierter Inhalt/effektive Absicht:

```yaml
ac-db-import:
  AC_PLAYERBOTS_DATABASE_INFO -> acore_playerbots

ac-worldserver:
  Playerbots enabled
  RandomBotAutologin = 1
  MinRandomBots = 1000
  MaxRandomBots = 1000
  PlayerLimit = 0
  MapUpdateThreads = 4
  SOAP enabled on internal port 7878
```

XP:

```text
Kill     2x
Quest    2x
Explore  2x
Pet      2x
```

Drops:

```text
Poor      2x
Normal    2x
Uncommon  2x
Rare      1.5x
Epic      1x
```

Mounts im Worldserver:

```text
./modules
  -> /azerothcore/modules:ro

./conf/dist/modules/playerbots.conf
  -> /azerothcore/env/dist/etc/modules/playerbots.conf:ro

./conf/dist/modules/mod_ahbot.conf
  -> /azerothcore/env/dist/etc/modules/mod_ahbot.conf:ro
```

Diese Mounts sind wichtig. Ihr Fehlen hat bereits einmal zu einem Worldserver ohne funktionierende Module geführt.

## 8. Playerbots

Config:

```text
conf/dist/modules/playerbots.conf
```

Wichtige dokumentierte Werte:

```text
AiPlayerbot.Enabled = 1
AiPlayerbot.RandomBotAutologin = 1

AiPlayerbot.MinRandomBots = 1500
AiPlayerbot.MaxRandomBots = 1500

AiPlayerbot.RandomBotAccountCount = 200
AiPlayerbot.AddClassAccountPoolSize = 50

AiPlayerbot.MaxAddedBots = 40
AiPlayerbot.BotAutologin = 0
AiPlayerbot.AllowAccountBots = 1
AiPlayerbot.AllowGuildBots = 1
AiPlayerbot.AllowTrustedAccountBots = 1
AiPlayerbot.AutoEquipUpgradeLoot = 1
```

### Wichtig: 1500 vs. 1000

Die Config-Datei enthält weiterhin 1500, aber das Docker-Override setzt:

```text
AC_AI_PLAYERBOT_MIN_RANDOM_BOTS=1000
AC_AI_PLAYERBOT_MAX_RANDOM_BOTS=1000
```

Im aktuellen Docker-Deployment sind deshalb **1000** die effektiven Werte.

Bei Zweifeln:

```bash
docker compose config | grep -E 'AC_AI_PLAYERBOT_(MIN|MAX)_RANDOM_BOTS'
```

Nicht allein `playerbots.conf` lesen und daraus 1500 aktive Bots ableiten.

### Verifizierter historischer Zustand

Nach der Cloud-Migration ergab:

```sql
SELECT COUNT(*) FROM acore_characters.characters WHERE online=1;
```

exakt 1000 Online-Charaktere. Playerbots war damit funktional aktiv.

Die Playerbot-DB wurde erfolgreich geöffnet und aktualisiert. Die Random-Bot-Tabelle dieser Revision heißt:

```text
playerbots_random_bots
```

Historischer DB-Stand nach Migration:

```text
15012 Rows in playerbots_random_bots
```

Das ist Diagnosekontext, kein Sollwert.

## 9. Playerbot-Level-Verteilung

Im ursprünglichen Setup wurde das Level-Bracket-System angepasst, um eine sinnvollere Verteilung der Bots über die Levelbereiche zu erhalten.

Historisch gewünschte statische Verteilung pro Fraktion:

```text
1–9     5%
10–19   8%
20–29  11%
30–39  12%
40–49  12%
50–59  13%
60–69  14%
70–79  13%
80      12%
```

Level-Brackets sollten aktiviert und dynamische Distribution deaktiviert sein.

**Wichtig:** Vor einer konkreten Änderung die aktuellen Property-Namen direkt in `conf/dist/modules/playerbots.conf` prüfen. Diese MEMORY-Datei soll keine möglicherweise versionsabhängigen Config-Schlüssel erfinden.

Historisch wurde außerdem `AiPlayerbot.RandomBotRpgChance = 0.80` verwendet, um mehr Bots mit RPG-Aktivität in der Welt zu sehen. Auch diesen Wert bei Bedarf gegen die aktuelle Config prüfen.

## 10. Playerbot-Companions / Altbots

Das Setup verwendet Trusted Account Linking statt Custom-Code.

Aktuelles Beispiel:

```text
Bot-Account:   domebot
Bot-Charakter: Leonard
```

Grundprinzip:

1. separaten Bot-Account erstellen
2. dort normalen Charakter erstellen
3. Bot-Account per Key mit Spieleraccount verknüpfen
4. Companion bei Bedarf explizit hinzufügen/entfernen

Commands:

```text
.playerbots account setKey KEY
.playerbots account link BOTACCOUNT KEY
.playerbots account linkedAccounts

.playerbots bot add BOTNAME
.playerbots bot remove BOTNAME
```

Die Verknüpfung ist **Account ↔ Account**, nicht Charakter ↔ Charakter.

Gewünschter Spielstil:

```text
Solo-Charakter -> Companion manuell per bot add verwenden
Coop-Charakter -> keinen Companion hinzufügen
```

Automatische charakterabhängige Companion-Zuordnung wurde bewusst **nicht implementiert**, um den Scope klein zu halten.

## 11. AHBot

Config:

```text
conf/dist/modules/mod_ahbot.conf
```

Dokumentierter Repository-Stand:

```text
AuctionHouseBot.EnableSeller = 1
AuctionHouseBot.EnableBuyer = 1
AuctionHouseBot.Account = 203
AuctionHouseBot.GUID = 2002
AuctionHouseBot.ItemsPerCycle = 200
AuctionHouseBot.ConsiderOnlyBotAuctions = 1
AuctionHouseBot.DuplicatesCount = 5
AuctionHouseBot.DivisibleStacks = 1
AuctionHouseBot.ProfessionItems = 1
```

Historischer AHBot-Charakter:

```text
Account ID 203
Character GUID 2002
Name Ahbotchar
```

**Diese IDs sind DB-spezifisch.** Bei einer frischen Installation neue IDs ermitteln und die Config aktualisieren.

SQL-Dateien der gepinnten AHBot-Version:

```text
modules/mod-ah-bot/data/sql/db-world/mod_auctionhousebot.sql
modules/mod-ah-bot/data/sql/db-world/auctionhousebot_professionItems.sql
modules/mod-ah-bot/data/sql/db-world/z_filter_disabled_and_trash.sql
```

Die Modul-Dokumentation verlangt manuellen Import in die passende Datenbank; hier ist das `acore_world`.

Historisch verwendete AH-Zielmengen:

```text
Alliance (AH 2): 2000–3000
Horde    (AH 6): 2000–3000
Neutral  (AH 7):  500–800
```

Diese Werte wurden in der Datenbank gesetzt und sind nicht allein durch `mod_ahbot.conf` reproduziert.

Vor SQL-Änderungen:

```sql
DESCRIBE acore_world.mod_auctionhousebot;
SELECT auctionhouse,name,minitems,maxitems FROM acore_world.mod_auctionhousebot;
```

## 12. Datenbanken

Vier relevante AzerothCore-Datenbanken:

```text
acore_auth
acore_characters
acore_world
acore_playerbots
```

Vollbackup:

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

DB-Dump enthält Accounts/Charaktere und darf nicht ins Git-Repository.

## 13. Client-Daten

Benötigt:

```text
Cameras
dbc
maps
mmaps
vmaps
```

Docker-Volume im bisherigen Projekt:

```text
azerothcore-wotlk_ac-client-data
```

Historische Größe:

```text
ca. 3.1 GiB entpackt
ca. 1.2 GiB als tar.gz
```

Diese Daten gehören nicht in Git.

## 14. Realm

Historisch vor Migration:

```text
192.168.178.94
```

Nach Cloud-Migration wurde `acore_auth.realmlist` auf die öffentliche Serveradresse aktualisiert.

Historischer Cloud-Wert:

```text
address      159.195.198.128
localAddress 159.195.198.128
port         8085
```

Client:

```text
set realmlist 159.195.198.128
```

Bei zukünftigen Deployments IP/Hostname neu bestimmen. Diese Werte nicht blind kopieren.

## 15. Bekannter Cloud-Host zum Zeitpunkt des Setups

Provider: netcup

Historischer Zustand:

```text
Debian GNU/Linux 13 (trixie), 13.6
8 vCPU
AMD EPYC 9645
15 GiB RAM
~503 GiB Disk
Docker 29.6.1
Docker Compose 5.3.1
```

Auf demselben Host liefen außerdem Palworld und Valheim. Deshalb bei Performanceproblemen immer Gesamtlast des Hosts berücksichtigen.

Keine Swap-Partition war beim ursprünglichen Check aktiv. Bei RAM-Druck kann ein kleiner Swap als Sicherheitsnetz sinnvoll sein, aber Performanceprobleme nicht durch Swap "lösen".

## 16. Performance-Historie

Mit 1500 Bots und hoher RPG-Aktivität lief der Worldserver historisch ungefähr bei:

```text
~297% CPU
~5.05 GiB RAM
```

MySQL ungefähr:

```text
~13% CPU
~529 MiB RAM
```

Danach wurde auf 1000 Bots reduziert, primär weil Startgebiete zu voll wirkten, nicht weil 1500 grundsätzlich unspielbar waren.

Diese Zahlen sind nur Vergleichswerte. Bei Problemen immer aktuell messen:

```bash
docker stats --no-stream
free -h
swapon --show
```

## 17. Build-Regel

Playerbots und AHBot sind C++-Module.

Nach Core-/Moduländerungen:

```bash
docker compose build --no-cache ac-worldserver
docker compose up -d --force-recreate ac-worldserver
```

Nur `docker restart ac-worldserver` reicht nach Build- oder Mount-Änderungen nicht.

Nach reinen Environment-/Mount-Änderungen mindestens:

```bash
docker compose up -d --force-recreate ac-worldserver
```

## 18. Wichtige Diagnosebefehle

Status:

```bash
docker compose ps
```

Worldserver-Logs:

```bash
docker compose logs --tail=200 ac-worldserver
```

Playerbot-Logs:

```bash
docker compose logs ac-worldserver \
  | grep -iE 'playerbot|random bot|rndbot'
```

AHBot-Logs:

```bash
docker compose logs ac-worldserver \
  | grep -iE 'ahbot|auction'
```

Effektive Compose-Konfiguration:

```bash
docker compose config
```

Mounts:

```bash
docker inspect ac-worldserver \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Worldserver-Konsole:

```bash
docker attach ac-worldserver
```

Detach ohne Stop:

```text
Ctrl+P, Ctrl+Q
```

DB-Erreichbarkeit:

```bash
docker compose exec -T ac-database \
  mysql -uroot -p"$DOCKER_DB_ROOT_PASSWORD" -e 'SELECT 1;'
```

Online-Charaktere:

```bash
docker compose exec -T ac-database \
  mysql -uroot -p"$DOCKER_DB_ROOT_PASSWORD" acore_characters \
  -e 'SELECT COUNT(*) FROM characters WHERE online=1;'
```

Ports:

```bash
ss -lntp
```

Ressourcen:

```bash
docker stats --no-stream
free -h
swapon --show
```

## 19. Bereits gelöste Probleme

### A. Server lief ohne Playerbots

Symptom:

- Worldserver startet
- Playerbot-Kommandos fehlen
- `/azerothcore/modules/mod-playerbots` fehlt im Container

Ursache beim ersten Cloud-Deployment:

```text
docker-compose.override.yml fehlte im Clone
```

Dadurch fehlten Modul- und Config-Mounts.

Lösung:

- Override ins Repository aufnehmen/pullen
- Submodules prüfen
- Worldserver bauen
- Container recreaten

### B. Falsches/prebuilt Worldserver-Image

Anfangs lief ein normaler `master`-Worldserver ohne die gewünschten Module.

Lösung:

```bash
docker compose build --no-cache ac-worldserver
docker compose up -d --force-recreate ac-worldserver
```

Im Log muss der erwartete Playerbot-Fork/Branch erkennbar sein.

### C. `ac-db-import`: Permission denied

Betroffene Pfade:

```text
/azerothcore/env/dist/etc
/azerothcore/env/dist/logs
```

Lösung beim damaligen Host:

```bash
mkdir -p env/dist/etc env/dist/logs
chown -R 1000:1000 env/dist/etc env/dist/logs
```

UID/GID immer mit `.env` abgleichen.

### D. MySQL öffentlich erreichbar

Upstream-Compose veröffentlicht standardmäßig `${DOCKER_DB_EXTERNAL_PORT:-3306}:3306`.

Lösung:

```dotenv
DOCKER_DB_EXTERNAL_PORT=127.0.0.1:3306
```

Danach DB-Container recreaten und extern prüfen.

### E. Bind-Mount zeigt alte Config

Bei einer per Rename/Inode-Austausch gespeicherten Einzeldatei sah der laufende Container noch die alte Datei.

Lösung:

```bash
docker compose up -d --force-recreate ac-worldserver
```

### F. `playerbot rndbot stats` ohne sichtbare Ausgabe

Dieser Command wurde auf der migrierten Revision erkannt, lieferte aber keine hilfreiche sichtbare Ausgabe. Nicht als einzigen Healthcheck verwenden.

Bessere Checks:

- Playerbot-Startup-Logs
- DB-Verbindung zu `acore_playerbots`
- Online-Charakterzahl
- Ingame sichtbare Bots

## 20. Bekannte nichtkritische Warnungen

Historisch gesehen:

```text
Can't set process priority class, error: Permission denied
```

Das war im Container nicht kritisch.

Außerdem gab es Warnungen wegen fehlender `mod_ale.conf` / ALE-Properties. Der Worldserver konnte trotzdem starten. Bei zukünftigen Problemen Warnungen nicht automatisch als Root Cause behandeln; zuerst Fehler (`ERROR`, Shutdown, DB failure) und tatsächliches Verhalten korrelieren.

## 21. Sicherheit

Prioritäten:

```text
3306 nicht öffentlich
7878 nicht öffentlich, wenn SOAP nicht extern gebraucht wird
3724 und 8085 nur wie benötigt veröffentlichen
starkes DB-Root-Passwort
.env nicht committen
DB-Dumps nicht committen
Client-Daten nicht committen
```

Historisch war MySQL kurz öffentlich und der Root-Fallback war `password`. Bei einem bestehenden Server sollte deshalb sichergestellt sein, dass das Passwort inzwischen rotiert wurde. **Nicht annehmen, dass dies geschehen ist; prüfen.**

## 22. Git-Workflow

Remote des eigenen Forks:

```text
origin = https://github.com/PragmaticBeaver/azerothcore-wotlk.git
```

Historisch verwendetes Upstream:

```text
upstream = https://github.com/mod-playerbots/azerothcore-wotlk.git
```

Default-/Arbeitsbranch:

```text
Playerbot
```

Bei Updates:

```bash
git status
git pull origin Playerbot
git submodule update --init --recursive
```

Submodules sind gepinnt. Nicht ungeprüft `git submodule update --remote` verwenden und anschließend erwarten, dass alles weiterhin kompatibel ist.

## 23. Dokumentationsregel für zukünftige Hilfe

Wenn der Nutzer diese Datei in einem neuen Chat bereitstellt und ein Problem meldet:

1. Zuerst klären, ob das Problem auf dem bestehenden Server oder einem frischen Deployment auftritt.
2. Aktuellen Git-Stand/Submodule nicht aus dieser Datei erraten; bei Relevanz prüfen.
3. Bei Docker-Problemen zuerst `docker compose ps`, relevante Logs und `docker compose config` ansehen.
4. Bei Modulproblemen zusätzlich Mounts und Submodule prüfen.
5. Bei DB-Problemen die konkrete DB/Tabelle mit `SHOW`, `DESCRIBE` oder kleinen `SELECT`s verifizieren, bevor SQL vorgeschlagen wird.
6. Keine Tabellen- oder Config-Namen erfinden. Bei Unsicherheit Repository/Modulquelle prüfen.
7. Keine Secrets in Chat, Git oder Dokumentation verlangen.
8. Änderungen möglichst klein und einzeln durchführen und danach verifizieren.
9. Vor destruktiven DB-/Bot-Operationen Backup empfehlen.
10. Der Nutzer bevorzugt schrittweises Debugging statt riesiger unstrukturierter Anleitungen.

## 24. Weiterführende Dokumentation

Im Repository liegt zusätzlich:

```text
SETUP.md
```

Diese Datei enthält die reproduzierbare Installations-/Migrationsanleitung. Bei einem kompletten Neuaufbau zuerst `SETUP.md` verwenden; diese `MEMORY.md` dient primär als Kontext für Architektur, Entscheidungen und Troubleshooting.
