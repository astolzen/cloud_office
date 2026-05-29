# nextcloud_rpm — Nextcloud + OnlyOffice als native RPM-Installation

## Übersicht

Dieses Rollout installiert **Nextcloud** und den **OnlyOffice Document Server**
**ohne Container** direkt auf einem **RHEL 9 / AlmaLinux 9 / Rocky Linux 9**-System
als native systemd-Dienste.

Das Ergebnis entspricht funktional dem Vier-Container-Rollout (`vier_container`):
MariaDB, Valkey, Nextcloud (Apache + PHP) und OnlyOffice Document Server
laufen auf einem einzigen Host.

> **Hinweis für größere Deployments:** Bei höherer Last empfiehlt es sich,
> den Document Server auf einem separaten Host zu betreiben, um seinen
> PostgreSQL/RabbitMQ/nginx-Stack vom Nextcloud-Apache-Stack zu trennen.

---

## Voraussetzungen

- Frisches EL9-System (RHEL, AlmaLinux oder Rocky Linux), root- oder sudo-Zugang
- Mindestens 4 GB RAM (8 GB empfohlen — OnlyOffice ist speicherhungrig)
- Mindestens 40 GB Speicherplatz
- Ausgehende HTTPS-Verbindung für Paketdownloads und den OnlyOffice-Connector
- Auflösbarer Hostname (z.B. `nextcloud.beispiel.de`)

---

## Dienste-Übersicht

| Dienst                  | Verwaltet durch        | Standard-Port |
|-------------------------|------------------------|---------------|
| MariaDB 10.11           | systemd (`mariadb`)    | 3306          |
| Valkey                  | systemd (`valkey`)     | 6379          |
| PHP-FPM 8.2             | systemd (`php-fpm`)    | 9000 (Socket) |
| Apache (Nextcloud)      | systemd (`httpd`)      | 80 / 443      |
| PostgreSQL (OnlyOffice) | systemd (`postgresql`) | 5432          |
| RabbitMQ (OnlyOffice)   | systemd (`rabbitmq-server`) | 5672    |
| nginx (OnlyOffice)      | systemd (`nginx`)      | 80*           |
| OnlyOffice docservice   | systemd (`ds-docservice`) | —           |
| OnlyOffice converter    | systemd (`ds-converter`) | —            |

> \* **Port-Konflikt:** OnlyOffice-nginx und Apache belegen beide Port 80.
> Lösung: Document Server auf einen anderen Port umlegen (z.B. 8888) oder
> Apache auf 8080 verschieben. Siehe Abschnitt 9.

---

## 1. System vorbereiten

```bash
# Hostname setzen
hostnamectl set-hostname nextcloud.beispiel.de

# Vollständiges System-Update
dnf update -y

# EPEL und Basis-Werkzeuge
dnf install -y epel-release curl wget tar bzip2 unzip nano policycoreutils-python-utils

(Bei RHEL 9 läßt sich EPEL nicht direkt via dnf install epel-release einrichten. hier bitte
 dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm ausführen)

# SELinux-Status prüfen — muss auf Enforcing stehen
getenforce
```

> **SELinux bleibt im Enforcing-Modus.** Alle nötigen Policy-Anpassungen
> (`setsebool`, `semanage fcontext`) sind in den jeweiligen Abschnitten
> inline aufgeführt.

### Firewall-Ports öffnen

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

---

## 2. MariaDB (LTS)

```bash
curl -LsS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup \
  | bash -s -- --mariadb-server-version=mariadb-10.11

dnf install -y MariaDB-server MariaDB-client
systemctl enable --now mariadb
```

### Datenbank absichern und anlegen

```bash
mysql_secure_installation
# Root-Passwort vergeben, anonyme Nutzer entfernen, Remote-Root deaktivieren,
# Test-Datenbank entfernen
```

```bash
mysql -u root -p <<'SQL'
CREATE DATABASE nextcloud
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_general_ci;

CREATE USER 'nextcloud'@'localhost' IDENTIFIED BY 'BITTE_AENDERN_DB_PASS';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost';
FLUSH PRIVILEGES;
SQL
```

### MariaDB für Nextcloud optimieren

Datei `/etc/my.cnf.d/nextcloud.cnf` anlegen:

```ini
[mysqld]
transaction_isolation    = READ-COMMITTED
log_bin                  = binlog
binlog_format            = ROW
innodb_buffer_pool_size  = 256M
```

```bash
systemctl restart mariadb
```

---

## 3. Valkey (Redis-kompatibler Cache)

Valkey ist in EPEL für EL9 verfügbar.

```bash
dnf install -y valkey
systemctl enable --now valkey

# Verbindungstest
valkey-cli ping
# Erwartete Antwort: PONG
```

---

## 4. PHP 8.2 (Remi-Repository)

```bash
dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm
dnf module reset -y php
dnf module enable -y php:remi-8.2

dnf install -y \
  php php-cli php-fpm \
  php-gd php-json php-mbstring php-mysqlnd php-xml php-zip \
  php-curl php-intl php-bcmath php-gmp \
  php-pecl-imagick php-pecl-apcu php-pecl-redis \
  php-opcache php-process
```

### PHP konfigurieren

`/etc/php.ini` anpassen:

```ini
memory_limit            = 512M
upload_max_filesize     = 512M
post_max_size           = 512M
max_execution_time      = 360
date.timezone           = Europe/Berlin
opcache.enable          = 1
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files   = 10000
opcache.memory_consumption      = 128
opcache.save_comments           = 1
opcache.revalidate_freq         = 1
```

`/etc/php-fpm.d/www.conf` — User/Group auf Apache setzen:

```ini
user  = apache
group = apache
```

```bash
systemctl enable --now php-fpm
```

---

## 5. Apache (httpd)

```bash
dnf install -y httpd mod_ssl
systemctl enable --now httpd

# SELinux: Apache darf auf MariaDB und Valkey zugreifen
setsebool -P httpd_can_network_connect_db 1
setsebool -P httpd_can_network_connect 1
```

### Virtual-Host anlegen

`/etc/httpd/conf.d/nextcloud.conf`:

```apache
<VirtualHost *:80>
    ServerName nextcloud.beispiel.de
    DocumentRoot /var/www/html/nextcloud

    <Directory /var/www/html/nextcloud>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews

        <IfModule mod_dav.c>
            Dav off
        </IfModule>
    </Directory>

    RewriteEngine On
    RewriteRule ^/.well-known/carddav  /nextcloud/remote.php/dav  [R=301,L]
    RewriteRule ^/.well-known/caldav   /nextcloud/remote.php/dav  [R=301,L]

    ErrorLog  /var/log/httpd/nextcloud_error.log
    CustomLog /var/log/httpd/nextcloud_access.log combined
</VirtualHost>
```

```bash
httpd -M | grep rewrite   # mod_rewrite muss geladen sein
```

---

## 6. Nextcloud herunterladen und installieren

```bash
cd /tmp
curl -LO https://download.nextcloud.com/server/releases/latest.tar.bz2
curl -LO https://download.nextcloud.com/server/releases/latest.tar.bz2.sha256

sha256sum -c latest.tar.bz2.sha256

tar -xjf latest.tar.bz2 -C /var/www/html/
chown -R apache:apache /var/www/html/nextcloud
chmod -R 755 /var/www/html/nextcloud
```

### Daten-Verzeichnis anlegen (außerhalb des Webroot — empfohlen)

```bash
mkdir -p /var/nextcloud/data
chown -R apache:apache /var/nextcloud/data
chmod 750 /var/nextcloud/data
```

### SELinux-Dateikontexte setzen

```bash
semanage fcontext -a -t httpd_sys_rw_content_t "/var/nextcloud/data(/.*)?"
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/html/nextcloud/config(/.*)?"
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/html/nextcloud/apps(/.*)?"
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/html/nextcloud/apps-external(/.*)?"
restorecon -Rv /var/nextcloud/data
restorecon -Rv /var/www/html/nextcloud/config
restorecon -Rv /var/www/html/nextcloud/apps
systemctl reload httpd
```

---

## 7. Nextcloud-Installer ausführen

### Option A — Web-Installer

`http://nextcloud.beispiel.de/` im Browser öffnen und ausfüllen:

| Feld             | Wert                        |
|------------------|-----------------------------|
| Admin-Benutzername | `admin`                   |
| Admin-Passwort   | *(sicheres Passwort wählen)*|
| Daten-Ordner     | `/var/nextcloud/data`       |
| Datenbank         | MySQL/MariaDB              |
| DB-Host          | `localhost`                 |
| DB-Name          | `nextcloud`                 |
| DB-Benutzer      | `nextcloud`                 |
| DB-Passwort      | `BITTE_AENDERN_DB_PASS`     |

### Option B — Kommandozeilen-Installer

```bash
sudo -u apache php /var/www/html/nextcloud/occ maintenance:install \
  --database        "mysql" \
  --database-host   "localhost" \
  --database-name   "nextcloud" \
  --database-user   "nextcloud" \
  --database-pass   "BITTE_AENDERN_DB_PASS" \
  --admin-user      "admin" \
  --admin-pass      "BITTE_AENDERN_ADMIN_PASS" \
  --data-dir        "/var/nextcloud/data"
```

---

## 8. Nextcloud Post-Install-Konfiguration

```bash
# Trusted Domain setzen
sudo -u apache php /var/www/html/nextcloud/occ \
  config:system:set trusted_domains 0 --value="nextcloud.beispiel.de"

# Redis/Valkey als Session- und File-Locking-Store
sudo -u apache php /var/www/html/nextcloud/occ \
  config:system:set redis host --value="127.0.0.1"
sudo -u apache php /var/www/html/nextcloud/occ \
  config:system:set redis port --type=integer --value=6379
sudo -u apache php /var/www/html/nextcloud/occ \
  config:system:set memcache.locking --value="\OC\Memcache\Redis"
sudo -u apache php /var/www/html/nextcloud/occ \
  config:system:set memcache.distributed --value="\OC\Memcache\Redis"
sudo -u apache php /var/www/html/nextcloud/occ \
  config:system:set memcache.local --value="\OC\Memcache\APCu"

# Empfohlene Tweaks
sudo -u apache php /var/www/html/nextcloud/occ \
  config:system:set default_phone_region --value="DE"
sudo -u apache php /var/www/html/nextcloud/occ \
  config:system:set maintenance_window_start --type=integer --value=1
```

### Cron-Job einrichten

```bash
sudo -u apache php /var/www/html/nextcloud/occ background:cron

echo "*/5 * * * * apache php -f /var/www/html/nextcloud/cron.php" \
  > /etc/cron.d/nextcloud
```

---

## 9. OnlyOffice Document Server (RPM)

OnlyOffice Document Server bringt seine eigenen Dienste mit (PostgreSQL,
RabbitMQ, nginx). Diese laufen unabhängig von Nextclouds MariaDB und Apache.

### OnlyOffice-Repository hinzufügen

```bash
curl -LO https://download.onlyoffice.com/repo/centos/main/noarch/onlyoffice-repo.noarch.rpm
rpm -i onlyoffice-repo.noarch.rpm
```

### Abhängigkeiten installieren

```bash
dnf install -y postgresql-server rabbitmq-server nginx
```

> **Port-Konflikt zwischen Apache und OnlyOffice-nginx:**
> Beide belegen standardmäßig Port 80. Lösungsmöglichkeiten:
>
> - Document Server auf einen anderen Port umlegen (empfohlen für Single-VM):
>   Eintrag in `/etc/onlyoffice/documentserver/nginx/ds.conf` anpassen,
>   z.B. `listen 8888;`
> - Apache auf Port 8080 verschieben (Virtual-Host anpassen)
> - Separaten Host/IP für den Document Server verwenden

### Document Server installieren

```bash
dnf install -y onlyoffice-documentserver
```

Das Post-Install-Skript führt `documentserver-configure.sh` automatisch aus.
Falls nicht:

```bash
/usr/bin/documentserver-configure.sh
```

Dieses Skript initialisiert die interne PostgreSQL-Datenbank und generiert
JWT-Secrets.

### Dienste aktivieren und starten

```bash
systemctl enable --now rabbitmq-server
systemctl enable --now postgresql
systemctl enable --now ds-converter
systemctl enable --now ds-docservice
systemctl enable --now ds-metrics
systemctl enable --now nginx   # OnlyOffice-eigener nginx
```

### JWT-Secret setzen

`/etc/onlyoffice/documentserver/local.json` bearbeiten — den `token`-Block
anpassen:

```json
{
  "services": {
    "CoAuthoring": {
      "token": {
        "enable": {
          "request": { "inbox": true, "outbox": true },
          "browser": true
        },
        "inbox":  { "header": "Authorization" },
        "outbox": { "header": "Authorization" }
      },
      "secret": {
        "inbox":   { "string": "BITTE_AENDERN_JWT_SECRET" },
        "outbox":  { "string": "BITTE_AENDERN_JWT_SECRET" },
        "session": { "string": "BITTE_AENDERN_JWT_SECRET" }
      }
    }
  }
}
```

Änderung übernehmen:

```bash
supervisorctl restart all
# oder:
systemctl restart ds-docservice ds-converter
```

### Document Server prüfen

```bash
curl http://localhost/healthcheck
# Erwartete Antwort: true
```

---

## 10. OnlyOffice-Connector in Nextcloud installieren und konfigurieren

```bash
# Connector-App installieren
sudo -u apache php /var/www/html/nextcloud/occ app:install onlyoffice
sudo -u apache php /var/www/html/nextcloud/occ app:enable onlyoffice

# Document-Server-URL (Browser-seitig)
sudo -u apache php /var/www/html/nextcloud/occ \
  config:app:set onlyoffice DocumentServerUrl \
  --value="http://<DOCUMENT_SERVER_URL>/"

# Document-Server interne URL (Server-zu-Server)
sudo -u apache php /var/www/html/nextcloud/occ \
  config:app:set onlyoffice DocumentServerInternalUrl \
  --value="http://<DOCUMENT_SERVER_URL>/"

# Storage-URL (Document Server ruft Dateien hier ab)
sudo -u apache php /var/www/html/nextcloud/occ \
  config:app:set onlyoffice StorageUrl \
  --value="http://nextcloud.beispiel.de/"

# JWT-Einstellungen — müssen mit local.json auf dem Document Server übereinstimmen
sudo -u apache php /var/www/html/nextcloud/occ \
  config:app:set onlyoffice jwt_enabled --value="true"
sudo -u apache php /var/www/html/nextcloud/occ \
  config:app:set onlyoffice jwt_secret \
  --value="BITTE_AENDERN_JWT_SECRET"
sudo -u apache php /var/www/html/nextcloud/occ \
  config:app:set onlyoffice jwt_header --value="Authorization"
```

### Integration prüfen

```bash
sudo -u apache php /var/www/html/nextcloud/occ onlyoffice:documentserver --check
# Erwartete Ausgabe: Document Server is available
```

---

## 11. Abschließende Prüfung

```bash
# Nextcloud-Gesamtstatus
sudo -u apache php /var/www/html/nextcloud/occ status
sudo -u apache php /var/www/html/nextcloud/occ check
sudo -u apache php /var/www/html/nextcloud/occ setupchecks

# Hintergrundjobs einmalig ausführen (bereinigt Setup-Warnungen)
sudo -u apache php /var/www/html/nextcloud/occ background:cron
```

---

## Passwort zurücksetzen

```bash
sudo -u apache OC_PASS="neues_passwort" \
  php /var/www/html/nextcloud/occ user:resetpassword --password-from-env admin
```
