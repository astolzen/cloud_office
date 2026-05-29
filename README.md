# cloud_office – Nextcloud & OnlyOffice Rollout-Sammlung

Dieses Repository enthält Ansible-Playbooks und Schritt-für-Schritt-Anleitungen
für verschiedene Varianten des automatisierten Rollouts von
**Nextcloud** und **OnlyOffice** mit **Podman** auf Enterprise-Linux-Systemen.

Alle Rollouts sind voneinander unabhängig und können je nach Anforderung
(Isolation, Ressourcen, Netzwerkmodell) einzeln eingesetzt werden.

---

## Übersicht der Rollout-Varianten

### 1. [`vier_container`](./vier_container/README.md) — Vier eigenständige Podman-Container

Nextcloud (Apache), OnlyOffice Document Server, MariaDB und Valkey laufen
als vier separate Podman-Container, jeder mit einer eigenen externen IP-Adresse
im lokalen Netzwerk. Voraussetzung ist ein **gebridgtes Aardvark-Netzwerk**
in Podman, das den Containern eigene IP-Adressen aus dem LAN-Bereich zuweist.

**Persistenz:** Unterverzeichnisse unter `/var/pods/nextcloud/` auf dem Host.

**Besonderheit:** Maximale Netzwerk-Isolation, jeder Dienst ist direkt per
eigener IP erreichbar.

---

### 2. [`ein_pod`](./ein_pod/README.md) — Alle Dienste in einem Podman-Pod

Alle vier Dienste (Nextcloud, OnlyOffice, MariaDB, Valkey) laufen in einem
einzigen Podman-Pod auf dem Host-Netzwerk. Der Pod exportiert
Port **8088 → 80** (Nextcloud) und Port **8080 → 8080** (OnlyOffice).
Die Container kommunizieren intern über `127.0.0.1` im gemeinsamen
Pod-Netzwerk-Namespace.

**Persistenz:** Unterverzeichnisse unter `/var/pods/nextcloud_pod/` auf dem Host.

**Besonderheit:** Kein separates Bridge-Netzwerk nötig, einfachere
Netzwerkkonfiguration.

---

### 3. [`nextcloud_rpm`](./nextcloud_rpm/README.md) — Native RPM-Installation auf Enterprise Linux 9

Nextcloud und OnlyOffice Document Server werden **ohne Container** direkt
auf einem RHEL 9 / AlmaLinux 9 / Rocky Linux 9 Host installiert.
Alle Dienste (MariaDB, Valkey, Apache, PHP-FPM, nginx, OnlyOffice) laufen
als native systemd-Dienste.

**Besonderheit:** Kein Podman erforderlich, direkte RPM-Paketverwaltung,
SELinux bleibt durchgehend im Enforcing-Modus.

---

### 4. [`only_containers`](./only_containers/README.md) — OnlyOffice Workspace allein (ohne Nextcloud)

Rollout der **ONLYOFFICE Workspace Community Edition** als drei eigenständige
Podman-Container (MySQL, Document Server, Community Server) mit je eigener
externer IP im Aardvark-Bridge-Netzwerk. Kein Nextcloud, sondern die
vollständige OnlyOffice-Suite als eigenständige Plattform.

**Persistenz:** Unterverzeichnisse unter `/var/pods/only_office/` auf dem Host.

**Besonderheit:** Community Server läuft mit `--privileged` und
`--cgroupns=host`, da systemd als PID 1 im Container benötigt wird.

---

## Gemeinsame Voraussetzungen

- Ansible-Controller mit Python und installierten Collections:
  ```bash
  ansible-galaxy collection install containers.podman community.general ansible.posix
  ```
- Zielsystem: Enterprise Linux 9 (RHEL, AlmaLinux, Rocky Linux) oder Fedora
- Podman auf dem Zielsystem installiert (für Rollouts 1, 2, 4)
- Für Rollouts 1 und 4: Podman-Netzwerk vom Typ **macvlan** auf einer Host-Bridge vorhanden
  (Einrichtung: [`podman_bridge_networking.md`](./podman_bridge_networking.md))

## Konfiguration

In den Playbooks und Variablendateien der jeweiligen Unterordner sind alle
umgebungsspezifischen Werte (IP-Adressen, Passwörter, Hostnamen, ZFS-Pool)
durch Platzhalter der Form `<PLATZHALTER>` oder `BITTE_AENDERN_*` ersetzt.
Diese müssen vor dem ersten Einsatz angepasst werden.

## Verzeichnisstruktur

```
cloud_office/
├── README.md               ← Diese Übersicht
├── vier_container/         ← Rollout 1: 4 eigenständige Container
│   ├── README.md
│   ├── nextcloud_create.yml
│   └── nextcloud_delete.yml
├── ein_pod/                ← Rollout 2: Ein Podman-Pod (Host-Network)
│   ├── README.md
│   ├── pod_create.yml
│   ├── pod_delete.yml
│   └── pod_vars.yml
├── nextcloud_rpm/          ← Rollout 3: Native RPM-Installation
│   └── README.md
└── only_containers/        ← Rollout 4: OnlyOffice Workspace allein
    ├── README.md
    ├── only_office_create.yml
    └── only_office_delete.yml
```
