# Satisfactory Dedicated Server auf Mac (Apple Silicon / M1) — Setup

> Anleitung um dieses Repo auf einem Mac mit Apple-Silicon-Chip (M1/M2/M3/M4) per Docker
> zu betreiben. Funktioniert ohne Claude — einfach die Befehle der Reihe nach ausführen.
>
> ✅ **End-to-End verifiziert** auf M1 (macOS 26.5, Docker Desktop 28.0.4): Build + FEX-Emulation +
> Server-Download + Serverstart + Beanspruchen + Spielerstellung + **erfolgreicher Client-Beitritt
> eines Freundes über Tailscale**. Server lief mit ~25 Tick (Max 30), 0 Crashes.
> Satisfactory-Serverversion 1.2.0 (CL-491125), läuft als x64-Binary via FEX auf ARM64.
>
> **Wie es funktioniert:** Der Satisfactory-Server gibt es nur als x86-64-Programm. Dieses
> Image enthält **FEX-Emu**, das x86-64 auf ARM64 emuliert. Auf dem Mac läuft Docker in einer
> ARM64-Linux-VM (Docker Desktop) — im Container ist es also ARM64 Linux, und FEX übersetzt
> das x86-Spiel zur Laufzeit. Doppelte Schicht (VM + Emulation) ⇒ funktioniert, ist aber
> langsamer als nativ. Für einen Server ohne Grafik in der Regel okay.

---

## 1. Voraussetzungen

- **Docker Desktop für Mac (Apple Silicon)** installiert und gestartet.
  Download: https://www.docker.com/products/docker-desktop/
- Docker-Desktop-Ressourcen (Settings → Resources):
  - **RAM: 10–12 GB einstellen** (Default ist oft 8 GB → erhöhen!). Begründung siehe
    „RAM-Planung" unten. Mind. 8 GB nur für den Build selbst.
  - **Disk: mind. ~30 GB frei** (Image ~8 GB + Server-Install ~3 GB + FEX-Build-Layer + Savegames).
- Ca. 30–60 Min Zeit für den ersten Build + Server-Download.

Prüfen ob Docker läuft:
```bash
docker info >/dev/null 2>&1 && echo "Docker läuft" || echo "Docker NICHT erreichbar — Docker Desktop starten"
```

> **Wichtig (Mac vs. Linux):** Auf dem Mac **kein `sudo`** vor `docker` benutzen!
> Die Skripte `build.sh`/`run.sh` aus dem Repo sind für Linux (Oracle Cloud) und nutzen
> `sudo` bzw. `screen` — auf dem Mac stattdessen die Befehle unten verwenden.

### RAM-Planung (wichtig bei 16-GB-Macs)

Der Satisfactory-Server skaliert mit **Fabrik-Komplexität**, nicht mit Spielerzahl:
Early Game ~4 GB, Mid Game 6–8 GB, Late Game 8–12 GB, Mega-Fabrik 12–16 GB+.
Dazu kommt etwas FEX-Emulations-Overhead (~+0,5–1 GB).

Auf einem **16-GB-Mac** teilen sich macOS (~4 GB) und die Docker-VM den Speicher:
- **3–4 Spieler, Early/Mid Game:** reicht — Docker-VM auf **10–12 GB** stellen.
- **Late Game / Mega-Fabrik:** wird eng (Risiko Swapping/Crashes) — dann eher 32 GB RAM.

Laufenden Verbrauch beobachten: `docker stats satisfactory-server`

---

## 2. Repo holen

Dieses Repo enthält bereits **alle** Dateien des Upstream-Projekts (basiert auf
`sa-shiro/Satisfactory-Dedicated-Server-ARM64-Docker`) **plus** diese Anleitung — du musst das
Upstream-Repo **nicht** separat klonen. Auf einem neuen Rechner einfach dieses Repo holen:
```bash
git clone https://github.com/JojoJohnDoe/satisfactory-server-mac-mini.git
cd satisfactory-server-mac-mini
```
> Falls du das Repo schon gepullt hast und in dem Ordner bist: **diesen Schritt überspringen**,
> mit Schritt 3 weitermachen.

## 3. Daten-Verzeichnisse anlegen

Hier landen Savegame und Server-Config (per Volume gemountet):
```bash
mkdir -p satisfactory config
chmod 777 satisfactory config init-server.sh
chmod +x init-server.sh
```

## 4. Image bauen (FEX wird aus Quellcode kompiliert — dauert!)

```bash
docker build --platform linux/arm64 -t satisfactory-arm64 .
```
- Dauer: ~20–40 Min beim ersten Mal (FEX-Compile mit LTO).
- Bei „Killed" während `ninja` ⇒ zu wenig RAM ⇒ in Docker Desktop RAM erhöhen und erneut bauen.

## 5. Server starten

```bash
docker compose up -d
```
Beim **ersten Start** lädt SteamCMD den Server herunter (~3 GB) — das dauert ein paar Minuten.
Logs live ansehen:
```bash
docker logs -f satisfactory-server
```
Fertig, sobald `Server API listening on 0.0.0.0:7777` und `Engine is initialized` im Log stehen.
Ein frisch gestarteter Server steht dann auf `WaitingToStart` (Auto-Pause) — das ist normal:
Er „spielt" erst, sobald er **beansprucht** ist und eine **Session läuft** (Schritt 6).

> **Wichtig:** Die [`docker-compose.yml`](docker-compose.yml) nutzt das **vorgebaute Image**
> `satisfactory-arm64` (kein `build:`-Abschnitt). `docker compose up` **baut nicht selbst** —
> deshalb zuerst Schritt 4 ausführen, sonst kommt „image not found".

Die [`docker-compose.yml`](docker-compose.yml) ist direkt nutzbar (nichts anzupassen) und konfiguriert:
- **Ports:** `7777/udp`, `7777/tcp`, `8888/tcp`
- **Volumes:** `./satisfactory` (Installation), `./config` (Saves), `./init-server.sh` (Entrypoint)
- **`ALWAYS_UPDATE_ON_START=true`** → Server-Auto-Update bei jedem Start (Abschnitt 10)
- **`EXTRA_PARAMS`** → Startargumente des Servers (Logging etc.; hier auch Ports per `-Port=` o. Ä. änderbar)
- **`restart: unless-stopped`** → Container kommt nach Reboot/Absturz von selbst wieder hoch
  (sofern Docker Desktop beim Login automatisch startet — empfehlenswert für einen Dauerserver)

## 6. Server beanspruchen & Spiel anlegen (Pflicht nach dem ersten Start)

Ohne diesen Schritt kann niemand beitreten (sonst `EncryptionTokenMissing`). Zwei Wege — einer reicht:

### Variante A — im Spiel-Client (einfach)
Hauptmenü → **Server Manager** → **Add Server** → `IP-DES-MAC:7777`:
1. Server erscheint als „Offline/Unclaimed".
2. **Admin-Passwort setzen** (beansprucht den Server) + Servernamen vergeben.
3. **Create Game** (neues Spiel) oder ein vorhandenes Savegame laden.

### Variante B — headless per HTTPS-API (ohne Spiel-Client, ideal für den mini)
Alles per `curl` gegen die API auf Port 7777 (`-k` wegen selbstsigniertem Zertifikat). `DEIN_PASSWORT` ersetzen:
```bash
API="https://localhost:7777/api/v1/"

# 1) InitialAdmin-Token holen (nur solange Server unbeansprucht)
TOKEN=$(curl -sk -X POST "$API" -H "Content-Type: application/json" \
  -d '{"function":"PasswordlessLogin","data":{"MinimumPrivilegeLevel":"InitialAdmin"}}' \
  | python3 -c 'import json,sys; print(json.load(sys.stdin)["data"]["authenticationToken"])')

# 2) Server beanspruchen (Name + Admin-Passwort setzen)
curl -sk -X POST "$API" -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"function":"ClaimServer","data":{"ServerName":"Satisfactory M1","AdminPassword":"DEIN_PASSWORT"}}'

# 3) Mit Admin-Passwort neu anmelden
ADMIN=$(curl -sk -X POST "$API" -H "Content-Type: application/json" \
  -d '{"function":"PasswordLogin","data":{"MinimumPrivilegeLevel":"Administrator","Password":"DEIN_PASSWORT"}}' \
  | python3 -c 'import json,sys; print(json.load(sys.stdin)["data"]["authenticationToken"])')

# 4) Neues Spiel anlegen (NewGameData muss bei v1.2 ALLE Felder enthalten)
curl -sk -X POST "$API" -H "Content-Type: application/json" -H "Authorization: Bearer $ADMIN" \
  -d '{"function":"CreateNewGame","data":{"NewGameData":{"SessionName":"M1-Save","MapName":"","StartingLocation":"","bSkipOnboarding":true,"GameModeSettings":{},"AdvancedGameSettings":{},"CustomOptionsOnlyForModding":{}}}}'
# Erfolg = HTTP 202 (leerer Body)

# Status prüfen
curl -sk -X POST "$API" -H "Content-Type: application/json" -H "Authorization: Bearer $ADMIN" \
  -d '{"function":"QueryServerState","data":{}}'
```
`isGameRunning: true` in der Statusantwort = Welt läuft, jetzt kann beigetreten werden.

## 7. Verbinden (Satisfactory-Client)

Immer über den **Server Manager** verbinden — **nicht** über „Join Game"/direkte IP (sonst `EncryptionTokenMissing`):

**Server Manager → Add Server** → Adresse:
- Vom selben Rechner: `127.0.0.1:7777`
- Aus dem LAN: lokale IP des Macs (`ipconfig getifaddr en0`)
- Für Freunde von außerhalb: **Tailscale** (Schritt 8) — keine Portfreigabe nötig

Sobald der Server **Online** angezeigt wird → **Join Game**.

## 8. Freunde per Tailscale verbinden (jeder mit eigenem Account)

So brauchen Freunde **keine** Portfreigabe/öffentliche IP — sie behalten ihren **eigenen** Tailscale-Account.
Der gemappte Docker-Port (`0.0.0.0:7777`) ist automatisch über die Tailscale-IP des Macs erreichbar (TCP+UDP).

**Auf dem Server-Mac (mini):**
1. Tailscale installieren + anmelden. Tailscale-IP herausfinden: `tailscale ip -4` (Form `100.x.y.z`).
2. (Empfohlen, wie beim Plex-Setup) Gerät **taggen** statt an deinen Personen-Account zu binden —
   dann läuft es dauerhaft (kein 180-Tage-Key-Ablauf) und Zugriff wird per ACL gesteuert.

**In der Tailscale-Admin-Konsole (Access controls):**
```jsonc
// tagOwners: Tag definieren
"tagOwners": { "tag:satisfactory": ["autogroup:admin"] }

// acls: geteilten Freunden NUR die Spielports erlauben (gilt für TCP & UDP)
{ "action": "accept", "src": ["autogroup:shared"], "dst": ["tag:satisfactory:7777,8888"] }
```
Tag aufs Gerät: *Machines* → Gerät → `⋯` → **Edit ACL tags** → `tag:satisfactory`.

**Gerät teilen:** *Machines* → Gerät → `⋯` → **Share** → E-Mail/Link an den Freund.
Getaggte Geräte **können** geteilt werden; der Tag wird nur auf der Empfänger-Seite ausgeblendet,
der Zugriff bleibt über deine ACL geregelt.

**Beim Freund:** Einladung annehmen (eigener Account) → Tailscale-Client „connected" → in Satisfactory
**Server Manager → Add Server → `100.x.y.z:7777`** (Tailscale-IP des minis, **nicht** der kurze Name).

## 9. Ports

| Port | Protokoll | Zweck |
|------|-----------|-------|
| 7777 | UDP + TCP | Game / Verbindung (ab Satisfactory 1.0 alles über 7777) |
| 8888 | TCP | HTTPS-API |

## 10. Betrieb / nützliche Befehle

```bash
docker compose up -d            # starten (Hintergrund)
docker compose down             # stoppen
docker logs -f satisfactory-server   # Logs
docker compose restart          # neu starten
docker stats satisfactory-server     # RAM/CPU live
```
Auto-Update beim Start: in `docker-compose.yml` `ALWAYS_UPDATE_ON_START=true|false`.

### Updates (neue Satisfactory-Version)

Zwei getrennte Dinge:

- **Spielserver** (App 1690800): wird **nicht** ins Image gebaut, sondern von SteamCMD geladen.
  Mit `ALWAYS_UPDATE_ON_START=true` prüft `init-server.sh` bei **jedem Start** auf die neueste
  Version. Upgrade = einfach neu starten:
  ```bash
  docker compose down && docker compose up -d   # oder: docker compose restart
  ```
  → kann nicht veralten, kein Rebuild nötig. (Experimental-Branch: in `init-server.sh` vor
  `validate` ein `-beta experimental` einfügen.)

- **Image / FEX-Emu**: Der Dockerfile pinnt FEX bewusst auf einen alten Commit (neuere brechen den
  Build). Das ist stabil und führt den aktuellen Server aus. Neu bauen **nur** wenn ein künftiges
  Game-Update Crashes verursacht (FEX-Inkompatibilität) oder du Basis-Image-Sicherheitsupdates willst:
  ```bash
  docker compose down
  docker build --no-cache --platform linux/arm64 -t satisfactory-arm64 .   # oder: sh build-nocache.sh
  docker compose up -d
  ```

## 11. Backup & Migration (z. B. auf den Mac mini)

Wichtig zu wissen, **welcher Ordner was ist**:
- `./satisfactory` = die Server-**Installation** (~2,8 GB). **Nicht** sichern — wird per SteamCMD neu geladen.
- `./config` = **Savegames + Servereinstellungen**. **Das ist das Wertvolle.**
  - Saves: `./config/FactoryGame/Saved/SaveGames/server/*.sav`
  - Servereinstellungen: `./config/FactoryGame/Saved/SaveGames/ServerSettings.7777.sav`

**Backup:** nur `./config` sichern.

**Umzug auf den Mac mini:** Auf dem mini Schritte 1–5 durchführen, Server **stoppen**
(`docker compose down`), dann den Inhalt von `./config` vom alten Mac in das `./config` des minis
kopieren, wieder `docker compose up -d`. Der Server lädt die vorhandene Session automatisch
(`autoLoadSessionName`) — Schritt 6 entfällt dann. Die `./satisfactory`-Installation **nicht** kopieren.

---

## Fehlerbehebung

- **`Cannot connect to the Docker daemon`** → Docker Desktop starten.
- **`sudo docker ...` schlägt fehl** → auf dem Mac `sudo` weglassen.
- **Build bricht mit `Killed` ab** → RAM in Docker Desktop erhöhen (≥8 GB).
- **`ERROR: I do not have read/write permissions to /satisfactory`** →
  `chmod 777 satisfactory config` erneut ausführen.
- **`ERROR! Failed to install app '1690800' (Missing configuration)` beim ersten Start** →
  harmlos/normal. SteamCMD scheitert beim allerersten Versuch; weil `ALWAYS_UPDATE_ON_START=true`
  gesetzt ist, wird die Installation sofort automatisch wiederholt und läuft dann durch.
  Deshalb `ALWAYS_UPDATE_ON_START=true` (zumindest für den Erststart) gesetzt lassen.
- **Server startet aber Client findet ihn nicht** → Firewall/Port 7777 (UDP+TCP) prüfen;
  bei Remote-Zugriff Portweiterleitung im Router einrichten (oder Tailscale, Abschnitt 11).
- **`EncryptionTokenMissing` / sofortiger Disconnect beim Beitreten** → der Client verbindet
  über **„Join Game"/direkte IP** statt über den **Server Manager**. Token wird nur über den
  Server Manager vergeben. Lösung: Hauptmenü → **Server Manager** → Server auswählen → erst wenn
  **Online** → **Join Game**. Voraussetzung: Server ist beansprucht **und** ein Spiel läuft (Abschnitt 6/10).
  (Die in manchen Guides genannten Ports 15000/15777 sind hier irrelevant — das betrifft nur Internet-Hosting.)
