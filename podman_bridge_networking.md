# Podman-Netzwerk: Macvlan auf einer Host-Bridge

Dieses Dokument beschreibt, wie ein Podman-Netzwerk vom Typ **macvlan**
eingerichtet wird, das den Containern echte IP-Adressen im lokalen LAN
zuweist. Dieses Netzwerk ist Voraussetzung für die Rollouts
[`vier_container`](./vier_container/README.md) und
[`only_containers`](./only_containers/README.md).

---

## Warum macvlan?

Ein normales Podman-Bridge-Netzwerk (`--driver bridge`) erzeugt ein isoliertes
internes Netzwerk mit NAT. Container sind dann von außen nur über
Port-Forwarding erreichbar — das reicht nicht, wenn jeder Container eine
eigene IP-Adresse im LAN haben soll.

Mit **macvlan** erhält jeder Container eine eigene MAC-Adresse und erscheint
als eigenständiges Gerät im Netzwerk. Er bekommt eine IP-Adresse direkt aus
dem LAN-Bereich, ohne NAT oder Port-Mapping, und ist von anderen Hosts
direkt ansprechbar.

---

## Voraussetzung: Linux-Bridge auf dem Host

Macvlan in Podman benötigt als `parent`-Interface eine **Linux-Bridge** —
kein physisches Ethernet-Interface direkt. Die Bridge muss einmalig auf dem
Zielsystem angelegt und mit dem LAN-Interface verbunden werden.

### Mit nmcli (NetworkManager, persistent)

```bash
# Bridge anlegen
nmcli connection add type bridge ifname br_podman con-name br_podman

# Physisches Interface als Bridge-Mitglied hinzufügen
nmcli connection add type ethernet ifname <PHYSISCHES_INTERFACE> \
  master br_podman

# Bridge aktivieren
nmcli connection up br_podman
```

### Mit ip-Kommandos (nicht persistent, zum Testen)

```bash
ip link add br_podman type bridge
ip link set <PHYSISCHES_INTERFACE> master br_podman
ip link set br_podman up
ip link set <PHYSISCHES_INTERFACE> up
```

---

## Podman-Netzwerk anlegen

Das macvlan-Netzwerk wird einmalig angelegt und bleibt erhalten:

```bash
podman network create \
  --driver macvlan \
  --subnet <LAN_SUBNETZ> \
  --gateway <LAN_GATEWAY> \
  -o parent=<BRIDGE_INTERFACE> \
  <NETZWERK_NAME>
```

Konkretes Beispiel:

```bash
podman network create \
  --driver macvlan \
  --subnet 192.168.2.0/24 \
  --gateway 192.168.2.1 \
  -o parent=br_podman \
  pub_net
```

Die resultierende Konfiguration (`podman network inspect pub_net`):

```json
{
  "name": "pub_net",
  "driver": "macvlan",
  "network_interface": "br_podman",
  "subnets": [
    { "subnet": "192.168.2.0/24", "gateway": "192.168.2.1" }
  ],
  "ipv6_enabled": false,
  "dns_enabled": false,
  "ipam_options": { "driver": "host-local" }
}
```

> **Hinweis:** `dns_enabled: false` ist bei macvlan-Netzwerken normal.
> Aardvark-DNS ist hier nicht aktiv. Die Container kommunizieren
> ausschließlich über die statischen IPs, die im Playbook vergeben werden.

---

## Netzwerk entfernen

```bash
podman network rm <NETZWERK_NAME>
```

> Achtung: Laufende Container müssen vorher gestoppt werden.
