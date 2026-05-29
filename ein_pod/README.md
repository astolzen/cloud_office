# ein_pod — Nextcloud + OnlyOffice in einem Podman-Pod

## Übersicht

Dieses Rollout deployt **Nextcloud** mit dem **OnlyOffice Document Server**
als vier Container in einem einzigen **Podman-Pod** auf dem **Host-Netzwerk**.
Der Pod exportiert zwei Ports direkt auf die IP des Host-Systems:

| Port (Host) | Port (Container) | Dienst                       |
|-------------|------------------|------------------------------|
| **8088**    | 80               | Nextcloud (Apache)           |
| **8080**    | 8080             | OnlyOffice Document Server   |

Die Container teilen sich den Netzwerk-Namespace des Pods und kommunizieren
intern über `127.0.0.1` — kein separates Bridge-Netzwerk erforderlich.

---

## Architektur

```
Host-System (<HOST_IP>)
    │
    ├─ :8088  → Pod (ncp-app)   Nextcloud (Apache, Port 80)
    │
    └─ :8080  → Pod (ncp-docs)  OnlyOffice Document Server (Port 8080)
                    │
                    │  gemeinsamer 127.0.0.1 (Pod-Netzwerk-Namespace)
                    ├─ :3306  ncp-mariadb   MariaDB LTS
                    │         /var/pods/nextcloud_pod/mariadb/
                    │
                    ├─ :6379  ncp-redis     Valkey (ephemeral)
                    │
                    ├─ :8080  ncp-docs      OnlyOffice Document Server
                    │         /var/pods/nextcloud_pod/docs_pgsql/
                    │         /var/pods/nextcloud_pod/docs_data/
                    │         /var/pods/nextcloud_pod/docs_logs/
                    │         /var/pods/nextcloud_pod/docs_lib/
                    │
                    └─ :80    ncp-app       Nextcloud (Apache)
                              /var/pods/nextcloud_pod/app_html/
```

### Warum wird der Document-Server-nginx umkonfiguriert?

Im Podman-Pod teilen sich alle Container **denselben Netzwerk-Namespace**.
Nextclouds Apache beansprucht Port 80, daher muss der nginx des Document
Servers auf einen anderen Port ausweichen. Das Playbook patcht
`/etc/nginx/conf.d/ds.conf` im laufenden Container und setzt den
Listen-Port von 80 auf 8080.

---

## OnlyOffice-Connector-Konfiguration

| Einstellung                  | Wert                                        | Genutzt von          |
|------------------------------|---------------------------------------------|----------------------|
| `DocumentServerUrl`          | `http://<HOST_IP>:8080/`                    | Browser              |
| `DocumentServerInternalUrl`  | `http://127.0.0.1:8080/`                    | Nextcloud (PHP)      |
| `StorageUrl`                 | `http://<HOST_IP>:8088/`                    | Document Server      |

---

## Persistente Speicherung

Alle persistenten Daten liegen in Unterverzeichnissen unter `/var/pods/nextcloud_pod/`
auf dem Host-System. Das Playbook legt diese Verzeichnisse beim ersten Lauf
automatisch an.

```
/var/pods/nextcloud_pod/
├── mariadb/       → /var/lib/mysql           (ncp-mariadb)
├── app_html/      → /var/www/html            (ncp-app)
├── docs_pgsql/    → /var/lib/postgresql      (ncp-docs)
├── docs_data/     → /var/www/onlyoffice/Data (ncp-docs)
├── docs_logs/     → /var/log/onlyoffice      (ncp-docs)
└── docs_lib/      → /var/lib/onlyoffice      (ncp-docs)
```

Valkey-Daten sind ephemeral (kein persistentes Volume).

Das Basisverzeichnis kann in `pod_vars.yml` über die Variable `data_dir`
angepasst werden (Standard: `/var/pods/nextcloud_pod`).

---

## Konfiguration anpassen

Alle umgebungsspezifischen Werte befinden sich in `pod_vars.yml`:

```yaml
# Host-IP oder Hostname des Zielsystems
app:
  trusted_domain: "<HOST_IP_ODER_HOSTNAME>"

# Persistenter Speicher — Basisverzeichnis auf dem Host
data_dir: "/var/pods/nextcloud_pod"  # nach Bedarf anpassen

# Passwörter — UNBEDINGT vor Produktiveinsatz ändern
mariadb:
  root_password: "BITTE_AENDERN_DB_ROOT"
  password:      "BITTE_AENDERN_DB_PASS"
app:
  admin_password: "BITTE_AENDERN_ADMIN_PASS"
jwt_secret: "BITTE_AENDERN_JWT_SECRET"
```

In `pod_create.yml` und `pod_delete.yml` muss `<ZIELHOST>` durch den
Ansible-Hostnamen des Zielsystems ersetzt werden.

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
ansible-playbook -i hosts.yml ein_pod/pod_create.yml
```

Das Playbook führt folgende Schritte aus:

1. Kernel-Parameter `vm.overcommit_memory=1` für Valkey setzen
2. Aktuelle Container-Images ziehen
3. Verzeichnisse unter `/var/pods/nextcloud_pod/` anlegen
4. Verzeichnis-Berechtigungen setzen
5. Podman-Pod auf Host-Netzwerk mit Ports 8088:80 und 8080:8080 erstellen
6. MariaDB starten, auf Verbindungsbereitschaft warten
7. Valkey starten
8. OnlyOffice Document Server starten (nginx initial auf Port 80)
9. Warten bis OnlyOffice auf Port 80 antwortet, dann nginx auf 8080 umstellen
10. Nextcloud starten (verbindet sich mit 127.0.0.1 für DB und Cache)
11. Warten bis Document Server auf Port 8080 antwortet
12. Warten bis `/status.php` HTTP 200 zurückgibt (Nextcloud-Installation fertig)
13. OnlyOffice-Connector installieren und aktivieren
14. Document-Server-URLs, StorageUrl, JWT konfigurieren
15. Nextcloud Post-Install-Tweaks (Telefonregion, Wartungsfenster, Cron)

Nach Abschluss ist Nextcloud unter `http://<HOST_IP>:8088/` erreichbar.

---

## Abbau (Daten bleiben erhalten)

```bash
ansible-playbook -i hosts.yml ein_pod/pod_delete.yml
```

Entfernt den Pod und alle enthaltenen Container. Die Daten unter `/var/pods/nextcloud_pod/` bleiben erhalten.
Ein erneutes Ausführen von `pod_create.yml` verbindet sich wieder mit der
bestehenden Datenbank und Nextcloud-Installation.

---

## Vollständiges Löschen und Neuinstallation

```bash
ansible-playbook -i hosts.yml ein_pod/pod_delete.yml

# Auf dem Zielsystem — Datenpersistenz löschen
rm -rf /var/pods/nextcloud_pod

ansible-playbook -i hosts.yml ein_pod/pod_create.yml
```

---

## Update (neue Images einspielen)

```bash
ansible-playbook -i hosts.yml ein_pod/pod_delete.yml
ansible-playbook -i hosts.yml ein_pod/pod_create.yml

# Nextcloud-Datenbank-Upgrade im Container
podman exec --user www-data ncp-app php occ upgrade
```

---

## Troubleshooting

Pod- und Container-Status prüfen:

```bash
podman pod ps
podman ps --pod
```

Nextcloud-Logs:

```bash
podman logs --tail 50 ncp-app
```

Document-Server-Logs:

```bash
podman logs --tail 50 ncp-docs
```

Document-Server-Healthcheck (direkt im Container):

```bash
podman exec ncp-docs curl -sf http://localhost:8080/healthcheck
# Erwartete Antwort: true
```

Nextcloud-Statusseite:

```bash
curl http://<HOST_IP>:8088/status.php
# Erwartete Antwort: {"installed":true,"maintenance":false,...}
```

OnlyOffice-Integration testen:

```bash
podman exec --user www-data ncp-app php occ onlyoffice:documentserver --check
```

OnlyOffice-Connector-Einstellungen anzeigen:

```bash
podman exec --user www-data ncp-app php occ config:app:list onlyoffice
```

Nextcloud-Diagnose:

```bash
podman exec --user www-data ncp-app php occ status
podman exec --user www-data ncp-app php occ check
podman exec --user www-data ncp-app php occ setupchecks
```

MariaDB-Verbindungstest:

```bash
podman exec ncp-mariadb mariadb-admin ping -h localhost -uroot -pBITTE_AENDERN_DB_ROOT
```
