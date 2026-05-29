# only_containers — OnlyOffice Workspace als eigenständige Podman-Container

## Übersicht

Dieses Rollout deployt die **ONLYOFFICE Workspace Community Edition** als drei
eigenständige Podman-Container, jeder mit einer eigenen externen IP-Adresse
im lokalen Netzwerk. **Kein Nextcloud** — es handelt sich um die vollständige
OnlyOffice-Suite als eigenständige Kollaborationsplattform.

| Container            | Rolle                                     | Standard-Port |
|----------------------|-------------------------------------------|---------------|
| `only-office-mysql`  | MySQL 8.4 (externe Datenbank)             | 3306          |
| `only-office-docs`   | OnlyOffice Document Server (Editor-API)   | 80            |
| `only-office-app`    | OnlyOffice Community Server (Workspace-UI)| 80            |

Beim ersten Aufruf von `http://<IP_OO_APP>/` wird auf `/Wizard.aspx`
weitergeleitet, wo Admin-E-Mail und Passwort gesetzt werden.

---

## Netzwerk-Voraussetzung: Macvlan-Netzwerk auf einer Host-Bridge

> **Wichtig:** Wie das `vier_container`-Rollout erfordert auch dieses Setup
> ein Podman-Netzwerk vom Typ **macvlan**. Damit erhalten die Container eigene
> MAC-Adressen und erscheinen als eigenständige Geräte im LAN.
> Eine Linux-Bridge (`br_podman` o.ä.) muss auf dem Host vorhanden sein.

Vollständige Beschreibung zur Netzwerk-Einrichtung:
siehe [`vier_container/README.md`](../vier_container/README.md#netzwerk-voraussetzung-macvlan-netzwerk-auf-einer-host-bridge).

```bash
podman network create \
  --driver macvlan \
  --subnet <LAN_SUBNETZ> \
  --gateway <LAN_GATEWAY> \
  -o parent=<BRIDGE_INTERFACE> \
  <NETZWERK_NAME>
```

---

## Architektur

```
LAN (<LAN_SUBNETZ>)  —  macvlan auf br_podman
    │
    ├─ <IP_OO_MYSQL>  only-office-mysql  MySQL 8.4
    │                 /var/pods/only_office/mysql/
    │
    ├─ <IP_OO_DOCS>   only-office-docs   Document Server
    │                 /var/pods/only_office/docs_pgsql/
    │                 /var/pods/only_office/docs_data/
    │                 /var/pods/only_office/docs_logs/
    │                 /var/pods/only_office/docs_lib/
    │
    └─ <IP_OO_APP>    only-office-app    Community Server
                      /var/pods/only_office/app_data/
                      /var/pods/only_office/app_logs/
```

### Besonderheiten des Community Servers

**`--privileged` und `--cgroupns=host` sind erforderlich**, weil:

- Der Community Server **systemd als PID 1** im Container ausführt
  (offiziell von ONLYOFFICE so dokumentiert). `--privileged` gibt systemd
  die nötigen cgroup-Rechte.
- Auf EL9/Fedora mit **cgroup v2** enthält `/proc/1/cgroup` im privaten
  cgroup-Namespace nur `0::/`, sodass das Startup-Skript das libpod-Stichwort
  nicht findet und abbricht. Mit `--cgroupns=host` ist der echte Host-cgroup-
  Pfad sichtbar, die Erkennung klappt, und systemd wird korrekt gestartet.

**MySQL 8.4 Strict Mode** lehnt die Zero-Date-Standardwerte im ONLYOFFICE-
Schema ab. MySQL wird daher mit dem permissiven `sql_mode`
`ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION` gestartet.

---

## Persistente Speicherung

Alle persistenten Daten liegen in Unterverzeichnissen unter `/var/pods/only_office/`
auf dem Host-System. Das Playbook legt diese Verzeichnisse beim ersten Lauf
automatisch an.

```
/var/pods/only_office/
├── mysql/         → /var/lib/mysql              (only-office-mysql)
├── docs_pgsql/    → /var/lib/postgresql         (only-office-docs)
├── docs_data/     → /var/www/onlyoffice/Data    (only-office-docs)
├── docs_logs/     → /var/log/onlyoffice         (only-office-docs)
├── docs_lib/      → /var/lib/onlyoffice         (only-office-docs)
├── app_data/      → /var/www/onlyoffice/Data    (only-office-app)
└── app_logs/      → /var/log/onlyoffice         (only-office-app)
```

Daten bleiben über Container-Neustarts und `only_office_delete.yml` hinweg erhalten.

Das Basisverzeichnis kann in `only_office_create.yml` über die Variable
`data_dir` angepasst werden (Standard: `/var/pods/only_office`).

---

## Konfiguration anpassen

In `only_office_create.yml` müssen im `vars:`-Block folgende Werte
angepasst werden:

```yaml
# Netzwerk
net: "<NETZWERK_NAME>"

# IP-Adressen und MAC-Adressen
mysql:
  ip:  "<IP_OO_MYSQL>"
  mac: "<MAC_OO_MYSQL>"
docs:
  ip:  "<IP_OO_DOCS>"
  mac: "<MAC_OO_DOCS>"
app:
  ip:  "<IP_OO_APP>"
  mac: "<MAC_OO_APP>"

# Persistenter Speicher — Basisverzeichnis auf dem Host
data_dir: "/var/pods/only_office"  # nach Bedarf anpassen

# Passwörter und Secrets — UNBEDINGT vor Produktiveinsatz ändern
mysql:
  root_password: "BITTE_AENDERN_MYSQL_ROOT"
  password:      "BITTE_AENDERN_MYSQL_PASS"
jwt_secret: "BITTE_AENDERN_JWT_SECRET"
```

---

## Deployment

Voraussetzungen auf dem Ansible-Controller:

```bash
ansible-galaxy collection install containers.podman community.general
```

Hosts-Datei anlegen (z.B. `hosts.yml`):

```yaml
all:
  hosts:
    <ZIELHOST>:
      ansible_host: <ZIELHOST_IP>
      ansible_user: root
```

Playbook ausführen:

```bash
ansible-playbook -i hosts.yml only_containers/only_office_create.yml
```

Das Playbook führt folgende Schritte aus:

1. Aktuelle Container-Images ziehen
2. ZFS-Datasets anlegen (idempotent)
3. Verzeichnis-Berechtigungen setzen
4. MySQL mit permissivem `sql_mode` starten
5. Auf MySQL-Bereitschaft warten, `sql_mode` auch zur Laufzeit setzen
6. Document Server starten, auf `/healthcheck` warten
7. Community Server mit `privileged + cgroupns=host + systemd=always` starten
8. Auf Monoserve-FastCGI-Socket warten
9. Bei leerer Datenbank: Schema-SQL-Dateien einspielen, Monoserve neu starten
10. Community-Server-API auf Verfügbarkeit prüfen

---

## Abbau (Daten bleiben erhalten)

```bash
ansible-playbook -i hosts.yml only_containers/only_office_delete.yml
```

Stoppt und entfernt alle drei Container. ZFS-Datasets bleiben unberührt.
Ein erneutes Ausführen von `only_office_create.yml` überspringt die
Schema-Initialisierung (Tabellen existieren bereits).

---

## Vollständiges Löschen und Neuinstallation

```bash
ansible-playbook -i hosts.yml only_containers/only_office_delete.yml

# Auf dem Zielsystem — Datenpersistenz löschen
rm -rf /var/pods/only_office

ansible-playbook -i hosts.yml only_containers/only_office_create.yml
```

---

## Update (neue Images einspielen)

```bash
ansible-playbook -i hosts.yml only_containers/only_office_delete.yml
ansible-playbook -i hosts.yml only_containers/only_office_create.yml
```

---

## Troubleshooting

Container-Status prüfen:

```bash
podman ps
```

Community-Server-Logs:

```bash
podman logs --tail 50 only-office-app
```

Systemd-Dienste im Community-Server-Container:

```bash
podman exec only-office-app systemctl status monoserve nginx
```

Document-Server-Healthcheck:

```bash
curl http://<IP_OO_DOCS>/healthcheck
# Erwartete Antwort: true
```

Community-Server-API:

```bash
curl http://<IP_OO_APP>/api/2.0/capabilities.json
# Erwartete Antwort: JSON mit "statusCode":200
```

Monoserve-Socket prüfen:

```bash
podman exec only-office-app ls /var/run/onlyoffice/
```

MySQL-Verbindungstest aus dem Community Server:

```bash
podman exec only-office-app \
  mysqladmin --defaults-extra-file=/etc/mysql/conf.d/client.cnf ping
```

nginx-Konfiguration prüfen:

```bash
podman exec only-office-app nginx -t
```

OnlyOffice-Anwendungslog:

```bash
podman exec only-office-app tail -50 /var/log/onlyoffice/nginx.error.log
```

---

## Bekannte Einschränkungen

- Der Community Server **erfordert `--privileged`** (offizielle ONLYOFFICE-
  Vorgabe), damit systemd cgroups im Container verwalten kann.
- Auf EL9/Fedora mit cgroup v2 ist **`--cgroupns=host` zwingend** nötig.
- MySQL 8.4 Strict Mode muss für das ONLYOFFICE-Schema auf
  `ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION` reduziert werden.
- Der Community Server benötigt nach dem Start **60–90 Sekunden**, bis er
  vollständig antwortet (systemd-Boot + Mono-JIT-Kompilierung beim ersten
  Request).
