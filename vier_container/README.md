# vier_container — Nextcloud + OnlyOffice als vier eigenständige Podman-Container

## Übersicht

Dieses Rollout deployt **Nextcloud** zusammen mit dem **OnlyOffice Document
Server** als vier voneinander unabhängige Podman-Container. Jeder Container
erhält eine eigene externe IP-Adresse aus dem lokalen Netzwerk.

| Container     | Rolle                             | Standard-Port |
|---------------|-----------------------------------|---------------|
| `nc-mariadb`  | MariaDB LTS (Nextcloud-Datenbank) | 3306          |
| `nc-redis`    | Valkey (Cache + File-Locking)     | 6379          |
| `nc-app`      | Nextcloud (Apache)                | 80            |
| `nc-docs`     | OnlyOffice Document Server        | 80            |

---

## Netzwerk-Voraussetzung

Dieses Setup erfordert ein Podman-Netzwerk vom Typ **macvlan** auf einer
Linux-Host-Bridge. Damit erhält jeder Container eine eigene IP-Adresse
direkt im LAN — ohne NAT oder Port-Forwarding.

Einrichtung und Hintergründe: siehe
[`podman_bridge_networking.md`](../podman_bridge_networking.md).

Das Netzwerk muss existieren, bevor das Playbook ausgeführt wird.

---

## Architektur

```
LAN (<LAN_SUBNETZ>)  —  macvlan auf br_podman
    │
    ├─ <IP_NC_MARIADB>  nc-mariadb   MariaDB LTS
    │                   /var/pods/nextcloud/mariadb/
    │
    ├─ <IP_NC_REDIS>    nc-redis     Valkey (ephemeral)
    │
    ├─ <IP_NC_APP>      nc-app       Nextcloud (Apache)
    │                   /var/pods/nextcloud/app_html/
    │
    └─ <IP_NC_DOCS>     nc-docs      OnlyOffice Document Server
                        /var/pods/nextcloud/docs_pgsql/
                        /var/pods/nextcloud/docs_data/
                        /var/pods/nextcloud/docs_logs/
                        /var/pods/nextcloud/docs_lib/
```

Die OnlyOffice-Integration wird vom Playbook automatisch konfiguriert:
- Connector-App aus dem Nextcloud Appstore installiert
- Document-Server-URL, JWT-Secret und Header via `occ` gesetzt

---

## Persistente Speicherung

Alle persistenten Daten liegen in Unterverzeichnissen unter `/var/pods/nextcloud/`
auf dem Host-System. Das Playbook legt diese Verzeichnisse beim ersten Lauf
automatisch an.

```
/var/pods/nextcloud/
├── mariadb/       → /var/lib/mysql          (nc-mariadb)
├── app_html/      → /var/www/html           (nc-app)
├── docs_pgsql/    → /var/lib/postgresql     (nc-docs)
├── docs_data/     → /var/www/onlyoffice/Data (nc-docs)
├── docs_logs/     → /var/log/onlyoffice     (nc-docs)
└── docs_lib/      → /var/lib/onlyoffice     (nc-docs)
```

Valkey-Daten sind ephemeral (kein persistentes Volume).

Das Basisverzeichnis kann in `nextcloud_create.yml` über die Variable
`data_dir` angepasst werden (Standard: `/var/pods/nextcloud`).

---

## Konfiguration anpassen

Vor dem ersten Einsatz müssen in `nextcloud_create.yml` folgende Werte
im `vars:`-Block angepasst werden:

```yaml
# Netzwerk
net: "<NETZWERK_NAME>"          # Name des Podman-Bridge-Netzwerks

# IP-Adressen und MAC-Adressen
mariadb:
  ip:  "<IP_NC_MARIADB>"
  mac: "<MAC_NC_MARIADB>"       # z.B. 52:54:00:xx:xx:xx

redis:
  ip:  "<IP_NC_REDIS>"
  mac: "<MAC_NC_REDIS>"

app:
  ip:             "<IP_NC_APP>"
  mac:            "<MAC_NC_APP>"
  trusted_domain: "<IP_NC_APP>"  # oder Hostname (FQDN)

docs:
  ip:  "<IP_NC_DOCS>"
  mac: "<MAC_NC_DOCS>"

# Persistenter Speicher — Basisverzeichnis auf dem Host
data_dir: "/var/pods/nextcloud"  # nach Bedarf anpassen

# Passwörter und Secrets — UNBEDINGT vor Produktiveinsatz ändern
mariadb:
  root_password: "BITTE_AENDERN_DB_ROOT"
  password:      "BITTE_AENDERN_DB_PASS"
app:
  admin_password: "BITTE_AENDERN_ADMIN_PASS"
jwt_secret: "BITTE_AENDERN_JWT_SECRET"
```

---

## Deployment

Voraussetzungen auf dem Ansible-Controller:

```bash
ansible-galaxy collection install containers.podman community.general ansible.posix
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
ansible-playbook -i hosts.yml vier_container/nextcloud_create.yml
```

Das Playbook führt folgende Schritte aus:

1. Kernel-Parameter `vm.overcommit_memory=1` für Valkey setzen
2. Aktuelle Container-Images ziehen
3. Verzeichnisse unter `/var/pods/nextcloud/` anlegen
4. Verzeichnis-Berechtigungen setzen
5. MariaDB starten, auf Verbindungsbereitschaft warten
6. Valkey und OnlyOffice Document Server starten
7. Nextcloud starten (Image-Entrypoint installiert Nextcloud automatisch)
8. Auf Document-Server-Healthcheck warten
9. Auf `/status.php` HTTP 200 warten (Nextcloud-Installation abgeschlossen)
10. OnlyOffice-Connector-App installieren und aktivieren
11. Document-Server-URLs, JWT-Secret und Header konfigurieren
12. Nextcloud-Post-Install-Tweaks (Telefonregion, Wartungsfenster, Cron)

Nach Abschluss ist Nextcloud unter `http://<IP_NC_APP>/` erreichbar.

---

## Abbau (Daten bleiben erhalten)

```bash
ansible-playbook -i hosts.yml vier_container/nextcloud_delete.yml
```

Stoppt und entfernt alle vier Container. Die Daten unter `/var/pods/nextcloud/` bleiben erhalten.
Ein erneutes Ausführen von `nextcloud_create.yml` verbindet sich wieder
mit der bestehenden Datenbank und Installation.

---

## Vollständiges Löschen und Neuinstallation

```bash
ansible-playbook -i hosts.yml vier_container/nextcloud_delete.yml

# Auf dem Zielsystem — Datenpersistenz löschen
rm -rf /var/pods/nextcloud

ansible-playbook -i hosts.yml vier_container/nextcloud_create.yml
```

---

## Update (neue Images einspielen)

```bash
ansible-playbook -i hosts.yml vier_container/nextcloud_delete.yml
ansible-playbook -i hosts.yml vier_container/nextcloud_create.yml

# Nextcloud-Datenbank-Upgrade im Container
podman exec --user www-data nc-app php occ upgrade
```

---

## OnlyOffice-Connector-Konfiguration

Das Playbook setzt automatisch drei URLs:

| Einstellung                  | Bedeutung                                         |
|------------------------------|---------------------------------------------------|
| `DocumentServerUrl`          | URL, über die der Browser den Editor lädt        |
| `DocumentServerInternalUrl`  | URL, die Nextcloud serverseitig für Callbacks nutzt |
| `StorageUrl`                 | URL, über die OnlyOffice Dateien in Nextcloud liest/schreibt |

JWT ist auf beiden Seiten (Document Server und Nextcloud-Connector)
aktiv. Das Secret muss identisch sein.

---

## Troubleshooting

Container-Status prüfen:

```bash
podman ps
```

Nextcloud-Logs:

```bash
podman logs --tail 50 nc-app
```

Document-Server-Healthcheck:

```bash
curl http://<IP_NC_DOCS>/healthcheck
# Erwartete Antwort: true
```

Nextcloud-Statusseite:

```bash
curl http://<IP_NC_APP>/status.php
# Erwartete Antwort: {"installed":true,"maintenance":false,...}
```

OnlyOffice-Integration testen:

```bash
podman exec --user www-data nc-app php occ onlyoffice:documentserver --check
```

Nextcloud-Konfiguration prüfen:

```bash
podman exec --user www-data nc-app php occ status
podman exec --user www-data nc-app php occ check
podman exec --user www-data nc-app php occ setupchecks
```
