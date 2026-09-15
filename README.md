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
