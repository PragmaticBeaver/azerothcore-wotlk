# README

## Playerbot erstellen & verknüpfen

**1. Bot-Account erstellen** – in der AzerothCore-Konsole:

```text
account create BOTACCOUNT PASSWORT
```

Danach einmal mit dem neuen Account in WoW einloggen, **Bot-Charakter erstellen** und wieder ausloggen.

**2. Bot-Account freigeben** – mit dem Bot-Charakter einloggen:

```text
.playerbots account setKey MEINKEY
```

Danach wieder ausloggen.

**3. Mit Spieleraccount verknüpfen** – mit deinem normalen Charakter einloggen:

```text
.playerbots account link BOTACCOUNT MEINKEY
```

Verknüpfung prüfen:

```text
.playerbots account linkedAccounts
```

**4. Bot verwenden**

Bot einloggen und übernehmen:

```text
.playerbots bot add BOTNAME
```

Bot wieder entfernen:

```text
.playerbots bot remove BOTNAME
```

Für dein aktuelles Beispiel sind also `BOTACCOUNT = domebot` und `BOTNAME = Leonard`.

Der **Key ist nur für die Account-Verknüpfung** relevant. Danach brauchst du im Alltag praktisch nur noch `bot add` und `bot remove`.

### Mehrere Bots und Charaktere

Ja, genau. Das ist einer der großen Vorteile unseres jetzigen Aufbaus.

Du kannst problemlos mehrere getrennte Paare anlegen, zum Beispiel:

```text
Dein Account
├── Solo-Druid   ↔ Leonard
├── Solo-Warrior ↔ Geralt
└── Coop-Paladin → kein Bot

Bot-Accounts
├── domebot      → Leonard
└── domebot2     → Geralt
```

Du erstellst einfach einen weiteren Bot-Account samt Charakter und verknüpfst den Account wie eben beschrieben. Danach entscheidet im Grunde **nur dein `.playerbots bot add BOTNAME`**, welchen Companion du gerade mitnimmst.

Wichtig ist nur: Die Verknüpfung selbst ist **Account ↔ Account**, nicht **Charakter ↔ Bot**. Playerbots weiß also nicht automatisch „Leonard gehört zu Charakter A“. Diese Zuordnung machst du praktisch selbst, indem du mit Charakter A immer `bot add Leonard` und mit Charakter B beispielsweise `bot add Geralt` benutzt.

Der Aufwand für einen neuen persönlichen Companion ist damit wirklich gering – Account erstellen, Charakter erstellen, einmal Accounts verknüpfen, fertig.

## Mit einem Altbot leveln und questen

Ein Altbot ist ein eigenständiger WoW-Charakter. Er hat seine eigene Rasse, Klasse, Fähigkeiten, Ausrüstung und sein eigenes Questlog. Spieler und Bot müssen deshalb **nicht dieselbe Rasse oder Klasse** haben. Eine Blutelfe kann beispielsweise problemlos gemeinsam mit einem Tauren-Bot leveln.

### Gemeinsame Quests

Wenn Spieler und Bot dieselbe Quest annehmen dürfen, kann der Bot sie gemeinsam mit dem Spieler erledigen. Playerbots reagiert dabei auf verschiedene Aktionen des Gruppenleiters: Nimmt der Spieler eine Quest an, versucht der Bot sie ebenfalls anzunehmen; spricht der Spieler mit einem Questgeber, kann der Bot seine abgeschlossenen Quests ebenfalls abgeben. Auch Dungeon-Portale und Mount-/Unmount-Aktionen können vom Bot mitvollzogen werden.

Nicht jede Quest ist jedoch für jeden Charakter verfügbar. **Rassen-, Klassen- und andere Questvoraussetzungen gelten auch für Bots.** Ein Tauren-Bot kann daher beispielsweise eine Blutelfen-spezifische Quest nicht einfach übernehmen. Er kann den Spieler weiterhin begleiten und beim Kämpfen helfen, hat diese Quest dann aber nicht selbst im Questlog.

Gerade in den Startgebieten können sich die Questverläufe deshalb unterscheiden. Sobald beide Charaktere Quests aus gemeinsamen Horde-Gebieten annehmen können, lässt sich der Großteil der normalen Levelreise gemeinsam spielen.

Nützliche Quest-Kommandos:

```text
quests
quests all
accept [quest]
accept *
drop [quest]
talk
```

`quests` zeigt den Queststatus des Bots. Mit `accept *` kann der Bot alle für ihn verfügbaren Quests beim ausgewählten Questgeber annehmen. `talk` lässt ihn mit dem ausgewählten NPC interagieren, beispielsweise um eine Quest abzugeben.

### Fähigkeiten und Klassenentwicklung

Die Klasse des Spielers hat keinen Einfluss darauf, welche Fähigkeiten der Bot erlernen kann. Ein Tauren-Schamane entwickelt sich weiterhin als Schamane, ein Priester als Priester usw.

Für langfristig gespielte Altbots ist besonders dieses Kommando wichtig:

```text
maintenance
```

`maintenance` lässt den Altbot unter anderem verfügbare Zauber und Skills lernen, Verbrauchsgüter auffüllen, Ausrüstung verzaubern und reparieren. Es empfiehlt sich, das Kommando gelegentlich nach mehreren Level-Ups oder dann zu verwenden, wenn der Bot längere Zeit nicht gepflegt wurde.

Zum Kontrollieren der vorhandenen Zauber:

```text
spells
```

Bei einem ausgewählten Klassentrainer stehen außerdem zur Verfügung:

```text
trainer
trainer learn
```

`trainer` zeigt, was der Bot beim ausgewählten Trainer lernen kann, und `trainer learn` lässt ihn die verfügbaren Fähigkeiten lernen.

### Talente und Ausrüstung

Die aktuelle Talent-Spezialisierung kann mit folgendem Kommando geprüft werden:

```text
talents
```

Verfügbare Spezialisierungen:

```text
talents spec list
```

Eine Spezialisierung festlegen:

```text
talents spec [SPEC]
```

Für die automatische Ausrüstung eines Altbots steht außerdem zur Verfügung:

```text
autogear [option]
```

Altbots sind für langfristige persönliche Begleiter gedacht. Im Gegensatz zu den automatisch erzeugten Random Bots sollte man bei ihnen deshalb gelegentlich selbst auf `maintenance`, Talente und Ausrüstung achten.

### Praktischer Ablauf beim gemeinsamen Leveln

Für einen persönlichen Companion reicht im Alltag normalerweise:

```text
.playerbots bot add BOTNAME
```

Danach gemeinsam questen und gelegentlich den Zustand des Bots prüfen:

```text
quests
maintenance
spells
talents
```

Am Ende der Session kann der Bot wieder ausgeloggt werden:

```text
.playerbots bot remove BOTNAME
```

Unterschiedliche Rassen und Klassen sind für dieses Spielprinzip kein Problem. Entscheidend ist lediglich, ob eine konkrete Quest für beide Charaktere verfügbar ist.
