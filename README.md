# README

## Playerbot erstellen & verknüpfen

**1. Bot-Account erstellen** – in der AzerothCore-Konsole:

```text
account create BOTACCOUNT PASSWORT
```

Danach mit dem Account einloggen, Bot-Charakter erstellen und wieder ausloggen.

**2. Bot-Account freigeben** – mit dem Bot-Charakter:

```text
.playerbots account setKey MEINKEY
```

**3. Mit Spieleraccount verknüpfen** – mit dem normalen Charakter:

```text
.playerbots account link BOTACCOUNT MEINKEY
.playerbots account linkedAccounts
```

**4. Bot verwenden**

```text
.playerbots bot add BOTNAME
.playerbots bot remove BOTNAME
```

Die Verknüpfung gilt **Account ↔ Account**, nicht Charakter ↔ Bot. Welcher Companion verwendet wird, bestimmst du über `bot add BOTNAME`.

## Mit Altbots leveln

Altbots sind eigenständige Charaktere mit eigener Rasse, Klasse, Ausrüstung, Fähigkeiten und Questlog. Spieler und Bot können unterschiedliche Rassen und Klassen haben.

Quests können gemeinsam erledigt werden, sofern beide Charaktere sie annehmen dürfen. Rassen- und klassenspezifische Voraussetzungen gelten auch für Bots.

Wichtige Quest-Kommandos:

```text
quests
accept *
talk
```

### Fähigkeiten, Talente & Ausrüstung

Nach einigen Level-Ups gelegentlich ausführen:

```text
maintenance
```

Dadurch lernt der Altbot verfügbare Fähigkeiten und Skills, füllt Verbrauchsgüter auf, verzaubert Ausrüstung und repariert sie.

Weitere nützliche Kommandos:

```text
spells
trainer
trainer learn
talents
talents spec list
talents spec [SPEC]
autogear [option]
```

Altbots sind für langfristige Begleiter gedacht. Daher gelegentlich `maintenance`, Talente und Ausrüstung prüfen.
