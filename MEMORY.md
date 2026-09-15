# MEMORY.md — LLM Operational Context

> PURPOSE: Compact machine-oriented context for future ChatGPT sessions working on this AzerothCore server.
> This file is NOT authoritative for mutable runtime state.
> Do not infer missing facts. Do not invent AzerothCore config keys, commands, SQL tables, paths, module behavior, versions, or runtime state.

## 0. SOURCE-OF-TRUTH POLICY

Priority, highest first:

1. `CURRENT_RUNTIME` — current command output, logs, database queries, container inspection.
2. `CURRENT_REPO` — files currently present on branch `Playerbot`.
3. `PINNED_SOURCE` — source code/docs at the exact submodule commits recorded by Git.
4. `VERIFIED_HISTORY` — observations previously confirmed on a running instance.
5. `HISTORICAL_CONTEXT` — useful previous state that may now be stale.
6. Previous-chat/model memory.

Rules for assistant:

- Never override a higher-priority source with a lower-priority source.
- Before troubleshooting mutable state, ask for or retrieve current evidence when available.
- Treat repository configuration as desired/static configuration, NOT proof of current runtime state.
- Treat IDs, IPs, counts, versions and resource usage as potentially instance-specific/stale unless currently verified.
- When a config file and Compose environment disagree, inspect `docker compose config`; effective Compose environment can override config-file values.
- When uncertain about a command/config key/table/module behavior, verify against current repo or pinned module source instead of guessing.
- Never request, store, echo, or invent passwords, account keys, tokens, or other secrets.
- `.env` is intentionally not committed.
- Do not recommend committing SQL dumps, extracted WoW client data, `.env`, or secrets.

Status vocabulary used below:

```text
CURRENT_REPO       directly verified in repository when this file was generated
PINNED_SOURCE      verified from source at the repository-pinned module revision
VERIFIED_HISTORY   previously observed on a running server; re-check before relying on it
INSTANCE_SPECIFIC  belongs to one deployed DB/server and is not portable to a fresh install
INTENT             desired behavior/design, not proof of implementation
NOT_IN_SCOPE       explicitly excluded for now
UNKNOWN            not sufficiently verified
```

---

## 1. PROJECT IDENTITY

```yaml
project:
  game: "World of Warcraft: Wrath of the Lich King"
  client_version: "3.3.5a"
  client_build: 12340
  purpose: "private server for ~1-2 real players with a populated bot world"

repository:
  status: CURRENT_REPO
  github: "PragmaticBeaver/azerothcore-wotlk"
  branch: "Playerbot"
  upstream_family: "mod-playerbots AzerothCore Playerbot fork"

known_paths:
  local_dev:
    status: HISTORICAL_CONTEXT
    path: "/home/dome/src/cloud-server-setup/wow-server/azerothcore-wotlk"
  deployed_server:
    status: VERIFIED_HISTORY
    path: "/media/data/src/azerothcore-wotlk"
```

Human-facing reproducible installation documentation: `SETUP.md`.

---

## 2. ARCHITECTURE

```yaml
services:
  ac-database:
    role: "MySQL database"
  ac-db-import:
    role: "AzerothCore/module DB updater/import"
  ac-authserver:
    role: "WoW authentication/realm service"
  ac-worldserver:
    role: "AzerothCore world server + compiled modules"
  ac-client-data-init:
    role: "client data initialization"

databases:
  - acore_auth
  - acore_characters
  - acore_world
  - acore_playerbots

client_data_required:
  - Cameras
  - dbc
  - maps
  - mmaps
  - vmaps
```

`CURRENT_REPO`: base Compose file is `docker-compose.yml`; deployment customizations are in `docker-compose.override.yml`.

Do not edit upstream/base Compose merely to express deployment-specific settings when the override or `.env` is appropriate.

---

## 3. PINNED MODULES

Verified from current Git tree when this file was generated:

```yaml
modules:
  mod-playerbots:
    status: CURRENT_REPO
    path: "modules/mod-playerbots"
    remote: "https://github.com/mod-playerbots/mod-playerbots.git"
    branch_hint: "master"
    pinned_commit: "b6696bdbd3740e575598d167d69f39f68cc0b907"

  mod-ah-bot:
    status: CURRENT_REPO
    path: "modules/mod-ah-bot"
    remote: "https://github.com/azerothcore/mod-ah-bot.git"
    branch_hint: "master"
    pinned_commit: "a680cc1c98290713e9b3d3289544af78e5186dc1"
```

Important distinction: `.gitmodules` may contain branch hints, but reproducibility is determined by the submodule commit recorded by the parent repository.

Useful verification:

```bash
git submodule status
git status
git branch --show-current
```

---

## 4. COMPOSE OVERRIDE — VERIFIED STATIC CONFIG

The following values were verified from `docker-compose.override.yml` when this file was generated:

```yaml
worldserver_environment:
  AC_AI_PLAYERBOT_ENABLED: "1"
  AC_AI_PLAYERBOT_RANDOM_BOT_AUTOLOGIN: "1"
  AC_AI_PLAYERBOT_MIN_RANDOM_BOTS: "1000"
  AC_AI_PLAYERBOT_MAX_RANDOM_BOTS: "1000"
  AC_PLAYER_LIMIT: "0"
  AC_MAP_UPDATE_THREADS: "4"

xp_rates:
  AC_RATE_XP_KILL: "2"
  AC_RATE_XP_QUEST: "2"
  AC_RATE_XP_EXPLORE: "2"
  AC_RATE_XP_PET: "2"

drop_rates:
  AC_RATE_DROP_ITEM_POOR: "2"
  AC_RATE_DROP_ITEM_NORMAL: "2"
  AC_RATE_DROP_ITEM_UNCOMMON: "2"
  AC_RATE_DROP_ITEM_RARE: "1.5"
  AC_RATE_DROP_ITEM_EPIC: "1"
```

Expected module/config mounts from the override:

```text
./modules
  -> /azerothcore/modules:ro

./conf/dist/modules/playerbots.conf
  -> /azerothcore/env/dist/etc/modules/playerbots.conf:ro

./conf/dist/modules/mod_ahbot.conf
  -> /azerothcore/env/dist/etc/modules/mod_ahbot.conf:ro
```

Diagnostic:

```bash
docker compose config

docker inspect ac-worldserver \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

After changing mounts or Compose configuration, recreate rather than merely restart:

```bash
docker compose up -d --force-recreate ac-worldserver
```

After C++/module changes, rebuild first:

```bash
docker compose build --no-cache ac-worldserver
docker compose up -d --force-recreate ac-worldserver
```

---

## 5. PLAYERBOTS

### 5.1 Effective target

```yaml
random_bots:
  desired_online_count:
    status: CURRENT_REPO
    value: 1000
    source: "Compose environment min/max"

  previously_verified_online_count:
    status: VERIFIED_HISTORY
    value: 1000
    verification: "SELECT COUNT(*) FROM acore_characters.characters WHERE online=1"
```

Important known discrepancy:

```yaml
playerbots_conf:
  AiPlayerbot.MinRandomBots: 1500
  AiPlayerbot.MaxRandomBots: 1500
  status: CURRENT_REPO

compose_override:
  min_random_bots: 1000
  max_random_bots: 1000
  status: CURRENT_REPO
```

This is intentional/known. In the Docker deployment the Compose environment is expected to override the file values. Verify actual effective configuration with `docker compose config` and runtime evidence before diagnosing bot count.

### 5.2 Other relevant playerbots.conf values

Verified from current repository configuration when generated:

```yaml
playerbots:
  random_bot_account_count: 200
  add_class_account_pool_size: 50
  bot_autologin: 0
  allow_account_bots: 1
  allow_guild_bots: 1
  allow_trusted_account_bots: 1
  auto_equip_upgrade_loot: 1
```

Known historical tuning:

```yaml
random_bot_rpg_chance:
  status: VERIFIED_HISTORY
  value: 0.80
```

Do not assume undocumented LevelBracket key names from memory. Inspect current `conf/dist/modules/playerbots.conf` or pinned `mod-playerbots` source before modifying bracket configuration.

### 5.3 Playerbot DB

```yaml
playerbot_database:
  database: acore_playerbots
  known_random_bot_table:
    status: VERIFIED_HISTORY
    name: playerbots_random_bots
```

Historical migration observation:

```text
SELECT COUNT(*) FROM acore_playerbots.playerbots_random_bots;
=> 15012
```

This count is historical and must not be treated as current.

### 5.4 Useful diagnostics

```bash
docker compose logs ac-worldserver \
  | grep -iE 'playerbot|random bot|rndbot' \
  | tail -100
```

```sql
SELECT COUNT(*) AS online_characters
FROM acore_characters.characters
WHERE online = 1;
```

Note: `online=1` counts online characters; in a normal private deployment this can include real players as well as bots. Do not blindly equate it to exact random-bot count while real players are online.

---

## 6. PERSONAL COMPANION BOTS

Design intent:

```yaml
companion_model:
  status: INTENT
  approach: "separate WoW account containing persistent Altbot character"
  automatic_character_binding: false
  custom_code: false
  usage: "manual playerbots add/remove commands"
```

Current example:

```yaml
companion:
  status: HISTORICAL_CONTEXT
  account: domebot
  character: Leonard
```

Trusted-account workflow used/documented for this Playerbot setup:

```text
# on bot account / bot character
.playerbots account setKey TEMP_KEY

# on controlling player account
.playerbots account link BOTACCOUNT TEMP_KEY
.playerbots account linkedAccounts

# use companion
.playerbots bot add BOTNAME
.playerbots bot remove BOTNAME
```

Important semantic note:

```text
Trusted linking is Account <-> Account.
It is NOT a persistent Character <-> Character mapping.
```

Therefore a player's solo character can manually add Leonard, while a coop character on the same player account simply does not add him.

If any of these commands fail after an update, verify command syntax against the pinned/current `mod-playerbots` source instead of inventing alternatives.

---

## 7. AUCTION HOUSE BOT

Module:

```yaml
module: mod-ah-bot
config: conf/dist/modules/mod_ahbot.conf
```

Verified current repository config includes:

```yaml
ahbot:
  enable_seller: 1
  enable_buyer: 1
  account:
    value: 203
    status: INSTANCE_SPECIFIC
  guid:
    value: 2002
    status: INSTANCE_SPECIFIC
  consider_only_bot_auctions: 1
  duplicates_count: 5
  divisible_stacks: 1
  profession_items: 1
```

`Account=203` and `GUID=2002` belong to the migrated/current DB instance. They MUST NOT be assumed valid on a fresh database.

Pinned `mod-ah-bot` source verifies manual World DB SQL files at:

```text
modules/mod-ah-bot/data/sql/db-world/mod_auctionhousebot.sql
modules/mod-ah-bot/data/sql/db-world/auctionhousebot_professionItems.sql
modules/mod-ah-bot/data/sql/db-world/z_filter_disabled_and_trash.sql
```

The module's own README states that its SQL must be imported manually into the appropriate DB and the core cleanly rebuilt.

Known auction-house IDs from the module SQL/config context:

```text
2 = Alliance
6 = Horde
7 = Neutral
```

Historical production target quantities:

```yaml
auction_targets:
  status: VERIFIED_HISTORY
  alliance: [2000, 3000]
  horde: [2000, 3000]
  neutral: [500, 800]
```

These quantities were DB state, not necessarily represented by Git config. Before changing them, inspect the current schema/data:

```sql
DESCRIBE acore_world.mod_auctionhousebot;
SELECT auctionhouse,name,minitems,maxitems
FROM acore_world.mod_auctionhousebot;
```

Never assume a fresh DB already contains the historical target values.

---

## 8. DATABASE / BACKUP MODEL

Relevant DBs:

```text
acore_auth
acore_characters
acore_playerbots
acore_world
```

Full-state backup pattern:

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

Historical migration artifact sizes only; NOT current requirements:

```yaml
migration_artifacts:
  sql_dump:
    status: VERIFIED_HISTORY
    filename: azerothcore-full.sql
    approx_size: 376M
  client_data_archive:
    status: VERIFIED_HISTORY
    filename: ac-client-data.tar.gz
    approx_compressed_size: 1.2G
    approx_unpacked_size: 3.1G
```

Never commit these artifacts.

A full migration requires BOTH database state and client-data volume, in addition to Git source/config.

---

## 9. NETWORK / SECURITY

Expected public WoW endpoints:

```yaml
ports:
  auth:
    protocol: tcp
    port: 3724
    exposure: public
  world:
    protocol: tcp
    port: 8085
    exposure: public
```

Expected private/local-only services:

```yaml
mysql:
  container_port: 3306
  desired_host_binding: "127.0.0.1:3306"
  status: INTENT

soap:
  container_port: 7878
  desired_host_binding: "127.0.0.1:7878"
  status: INTENT
```

Recommended `.env` settings:

```dotenv
DOCKER_DB_EXTERNAL_PORT=127.0.0.1:3306
DOCKER_SOAP_EXTERNAL_PORT=127.0.0.1:7878
```

Do NOT claim these bindings are currently active without runtime verification:

```bash
ss -lntp | grep -E ':(3306|3724|7878|8085)\b'
docker compose ps
```

Historical security event:

```yaml
mysql_public_exposure:
  status: VERIFIED_HISTORY
  event: "3306 was initially published publicly by default Compose mapping"
  remediation: "DOCKER_DB_EXTERNAL_PORT=127.0.0.1:3306"
  external_test_after_fix: "connection refused"
```

Historical concern:

```yaml
mysql_root_password:
  status: HISTORICAL_CONTEXT
  issue: "deployment previously used fallback/default password 'password'"
  current_value: UNKNOWN
  instruction: "Never assume it is still password; never ask user to paste it. Verify configuration securely if relevant."
```

SOAP was historically observed publicly exposed before hardening discussion. Current exposure is UNKNOWN until checked.

---

## 10. DEPLOYMENT INSTANCE — HISTORICAL CONTEXT

The following describes the known server during initial deployment. It may change and must not be treated as current without verification.

```yaml
host:
  status: VERIFIED_HISTORY
  provider: netcup
  os: "Debian GNU/Linux 13 (trixie), 13.6"
  arch: amd64
  cpu: "8 vCPU, AMD EPYC 9645"
  ram_gib: 15
  swap_at_initial_setup: "none"
  disk:
    device: /dev/vda4
    size_gib: 503
  docker: "29.6.1"
  docker_compose: "v5.3.1"
  git: "2.47.3"

server_public_ipv4:
  status: INSTANCE_SPECIFIC
  historical_value: "159.195.198.128"
```

Never assume the historical public IP is still correct. Query current deployment/network state before using it in commands or client instructions.

Other game servers historically co-located on host:

```text
Palworld
Valheim
```

This matters for resource analysis and port conflicts, but current container state must be checked.

---

## 11. REALM / CLIENT

```yaml
client:
  version: "WoW 3.3.5a"
  build: 12340
  historical_locale: deDE
```

Historical migrated realm configuration:

```yaml
realm:
  status: VERIFIED_HISTORY
  id: 1
  name: AzerothCore
  address: "159.195.198.128"
  localAddress: "159.195.198.128"
  port: 8085
```

This is instance-specific and may be stale. Current truth:

```sql
SELECT id,name,address,localAddress,localSubnetMask,port
FROM acore_auth.realmlist;
```

Client `realmlist.wtf` must point to a currently reachable realm address.

---

## 12. KNOWN FAILURE MODES / DEBUGGING KNOWLEDGE

### F1 — Core starts but Playerbot functionality is absent

Historical root cause: a prebuilt normal/master worldserver image or build without correct module integration.

Verify:

```bash
git branch --show-current
git submodule status
docker compose logs ac-worldserver | head -100
```

Expected source/build family: Playerbot branch, not an unrelated plain AzerothCore `master` build.

### F2 — Source contains modules but container cannot see them

Historical root cause: `docker-compose.override.yml` missing on deployed host, therefore module/config bind mounts absent.

Verify mounts with `docker inspect`. Recreate container after correcting Compose.

### F3 — `ac-db-import` permission denied

Historical root cause: repository/host directories owned by root while containers run with UID/GID 1000.

Known remediation used:

```bash
mkdir -p env/dist/etc env/dist/logs
chown -R 1000:1000 env/dist/etc env/dist/logs
```

Do not blindly use UID 1000 on a different host; first inspect `.env` / deployment UID/GID.

### F4 — Edited bind-mounted config appears unchanged in running container

Historical cause: file replacement (`sed -i` or editor atomic rename) changed inode while container retained old single-file bind mount.

Remediation:

```bash
docker compose up -d --force-recreate ac-worldserver
```

### F5 — Playerbot count appears inconsistent

Known static discrepancy: file says 1500, Compose environment says 1000. Check effective Compose and runtime before changing anything.

### F6 — Database disconnect after recreating DB container

Historical observation: auth/world temporarily lost DB connection during DB recreation. After DB became healthy, restarting/recreating dependent services restored connectivity.

### F7 — Nonfatal process priority warning

Historical log:

```text
Can't set process priority class, error: Permission denied
```

Observed in containerized worldserver; historically noncritical. Re-evaluate if accompanied by actual startup failure.

### F8 — ALE module/config warnings

Historical warnings referenced missing:

```text
/azerothcore/env/dist/etc/modules/mod_ale.conf
ALE.Enabled
TraceBack
AutoReload
BytecodeCache
ScriptPath
RequirePaths
RequireCPaths
AutoReloadInterval
```

Historically nonfatal. Current status UNKNOWN. Do not assume ALE is intentionally configured.

---

## 13. PERFORMANCE HISTORY

Historical only:

```yaml
1500_random_bots_test:
  status: VERIFIED_HISTORY
  worldserver_cpu: "~297%"
  worldserver_ram: "~5.05 GiB"
  database_cpu: "~13%"
  database_ram: "~529 MiB"
  host_cpu_idle: "~65.7%"
  conclusion_at_time: "functional, but bot target later reduced to 1000 because starting zones felt crowded"
```

Do not use these figures as current benchmarks. Use:

```bash
docker stats --no-stream
free -h
swapon --show
```

---

## 14. GAMEPLAY DESIGN / CURRENT SCOPE

```yaml
gameplay:
  real_players: "typically 1-2"
  xp_multiplier: 2
  loot_philosophy: "reduce grind without trivializing rare/epic loot"
  random_bot_target: 1000
  auction_house: "AHBot populated"
  personal_companions: true
  custom_auto_companion_code: false

not_in_scope:
  llm_bot_chat:
    status: NOT_IN_SCOPE
    examples: [Ollama, LLM chatter]
```

Current static loot multipliers are documented in section 4. Do not describe `Rate.Drop.Item.Referenced` as a dedicated quest-item rate without source verification; that interpretation was previously identified as unsafe/oversimplified.

Early-riding custom module was discussed but is NOT documented here as implemented. Do not assume `mod-early-mount` exists unless current repository inspection confirms it.

---

## 15. SAFE DIAGNOSTIC COMMANDS

Project status:

```bash
git status
git branch --show-current
git submodule status
docker compose config
docker compose ps
```

Worldserver logs:

```bash
docker compose logs --tail=200 ac-worldserver
```

Playerbot-focused logs:

```bash
docker compose logs ac-worldserver \
  | grep -iE 'playerbot|random bot|rndbot' \
  | tail -100
```

Mounts:

```bash
docker inspect ac-worldserver \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Resources:

```bash
docker stats --no-stream
free -h
swapon --show
```

Listening ports:

```bash
ss -lntp
```

Attach to worldserver console:

```bash
docker attach ac-worldserver
```

Detach without stopping container:

```text
Ctrl+P, Ctrl+Q
```

When database credentials are needed, use the deployment's secure `.env` value. Do not assume the historical default.

---

## 16. UPDATE / CHANGE RULES FOR FUTURE ASSISTANT

Before suggesting changes:

```text
1. Identify whether the problem is source/config/runtime/database/network/client.
2. Inspect the highest available source of truth.
3. Prefer one logical change at a time.
4. Preserve existing working Playerbot/AHBot integration.
5. Avoid replacing working config with guessed defaults.
6. Before DB schema changes, inspect schema first.
7. Before module updates, note pinned commit and take DB backup.
8. C++ or submodule source change => rebuild image.
9. Compose mount/environment change => recreate affected container.
10. Pure runtime uncertainty => gather logs/config/DB evidence before proposing invasive changes.
```

For repository updates:

```bash
git pull origin Playerbot
git submodule update --init --recursive
```

Do NOT automatically run `git submodule update --remote` as a normal deployment step; that changes pinned module versions and can reduce reproducibility.

---

## 17. REPRODUCIBILITY BOUNDARY

Git is authoritative for:

```text
source tree
pinned submodule commits
Compose definitions/override
committed Playerbot configuration
committed AHBot configuration
committed XP/drop-rate configuration
SETUP.md / MEMORY.md
```

Git is NOT authoritative for:

```text
.env / secrets
current public IP
current container state
current resource usage
accounts / password hashes
characters / inventories / progress
AHBot instance account/GUID on a fresh DB
current auctions
current Playerbot DB state
realm DB address
client-data volume
DB-only AH quota modifications
```

Exact restore therefore needs:

```text
Git checkout + pinned submodules
+ .env/secrets supplied securely
+ database backup of all four DBs
+ ac-client-data volume/archive
```

Fresh install can instead initialize new databases/client data and follow `SETUP.md`, but instance-specific IDs and DB tuning must then be recreated.

---

## 18. FINAL ANTI-HALLUCINATION CHECKLIST

Before answering a server-specific technical question, future assistant should ask internally:

```text
[ ] Is this fact static repo state or mutable runtime state?
[ ] Do I have current evidence, or only historical context?
[ ] Is the exact command/config key/table verified for this Playerbot/AzerothCore revision?
[ ] Am I confusing upstream AzerothCore behavior with mod-playerbots fork behavior?
[ ] Am I confusing config-file values with Compose environment overrides?
[ ] Am I treating an instance-specific DB ID/IP as portable configuration?
[ ] Would inspecting current repo/logs/schema be safer than guessing?
[ ] Does this change require rebuild, recreate, restart, or only config reload?
[ ] Could this expose MySQL/SOAP or leak a secret?
```

If any answer is uncertain, verify first.
