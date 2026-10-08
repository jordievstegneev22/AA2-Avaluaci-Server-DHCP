# Servidor DHCP amb Kea

Pràctica UD2. AA2 — Serveis de Xarxa (SMX)

## Què he de fer

He de muntar un servidor DHCP en un Ubuntu Server.

Després he de comprovar amb un client Zorin que rep bé la IP, la porta d'enllaç i el DNS.

També he de gravar la negociació amb el Wireshark i fer una reserva d'IP.

El meu número de llista és el **10**. Per això faig servir la xarxa `192.169.10.0/24`.

## Les dues màquines

**Servidor:** Ubuntu 24.04 Server. Es diu `srv-smx01`.

**Client:** Zorin OS.

El servidor té dues targetes de xarxa:

- `enp0s3` està en **NAT**. Serveix per sortir a Internet.
- `enp0s8` està en **xarxa interna `SMX-LAB`**. Té la IP `192.169.10.1`. És la que fa servir el DHCP.

A la segona no hi poso porta d'enllaç ni DNS. Ho diu l'enunciat.

El client només té una targeta. Primer la poso en NAT. Després la canvio a la xarxa interna.

---

## 1. Configurar la xarxa del servidor

Obro el fitxer de netplan:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Hi escric això:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      addresses: [192.169.10.1/24]
```

Ho guardo i ho aplico:

```bash
sudo netplan apply
ip a
```

Amb `ip a` miro que `enp0s8` tingui la IP `192.169.10.1`.

---

## 2. Instal·lar el Kea

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install kea
```

Mentre s'instal·la surt una pantalla blava. Em demana una contrasenya.

Trio l'opció **`configured_random_password`**.

Jo treballo directament al servidor. No el faig servir des de fora. Per això aquesta contrasenya no m'importa.

Si m'equivoco puc tornar a veure la pantalla amb:

```bash
sudo dpkg-reconfigure kea-ctrl-agent
```

El Kea instal·la quatre serveis. Van per separat:

| Servei | El seu fitxer |
|---|---|
| `kea-dhcp4-server` | `/etc/kea/kea-dhcp4.conf` |
| `kea-dhcp6-server` | `/etc/kea/kea-dhcp6.conf` |
| `kea-dhcp-ddns-server` | `/etc/kea/kea-dhcp-ddns.conf` |
| `kea-ctrl-agent` | `/etc/kea/kea-ctrl-agent.conf` |

---

## 3. Apagar el que no faig servir

Jo només vull el DHCP per a IPv4.

Per tant apago els altres dos:

```bash
sudo systemctl stop kea-dhcp6-server
sudo systemctl disable kea-dhcp6-server

sudo systemctl stop kea-dhcp-ddns-server
sudo systemctl disable kea-dhcp-ddns-server
```

**Compte amb el nom del DDNS.** Es diu `kea-dhcp-ddns-server`.

A classe surt escrit `kea-dhcp-ddns`. Amb aquest nom no funciona. Em va sortir aquest error:

```
Failed to stop kea-dhcp-ddns.service: Unit kea-dhcp-ddns.service not loaded.
```

Ara ho comprovo:

```bash
systemctl list-unit-files | grep kea
```

El dhcp6 i el ddns han de sortir com a **disabled**.

El dhcp4 ha de sortir com a **enabled**.

![Els serveis del Kea](img/01-serveis-kea.png)

---

## 4. Escriure la configuració

Primer canvio el nom del fitxer que ve de sèrie. Així no el perdo:

```bash
cd /etc/kea
sudo mv kea-dhcp4.conf old-kea-dhcp4.conf
sudo nano kea-dhcp4.conf
```

El fitxer és en format JSON.

Això vol dir que els blocs van entre `{ }` i les llistes entre `[ ]`.

També vol dir que he de mirar molt bé on va coma i on no.

```json
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": [ "enp0s8" ]
    },
    "valid-lifetime": 4000,
    "renew-timer": 1000,
    "rebind-timer": 2000,
    "lease-database": {
      "type": "memfile",
      "persist": true,
      "name": "/var/lib/kea/kea-leases4.csv"
    },
    "subnet4": [
      {
        "id": 1,
        "subnet": "192.169.10.0/24",
        "option-data": [
          {
            "name": "routers",
            "data": "192.169.10.254"
          },
          {
            "name": "domain-name-servers",
            "data": "8.8.8.8"
          }
        ],
        "pools": [ { "pool": "192.169.10.10 - 192.169.10.50" } ]
      }
    ]
  }
}
```

Això és el que demana l'enunciat:

| Què | Valor |
|---|---|
| Pool | de 192.169.10.10 a 192.169.10.50 |
| Porta d'enllaç | 192.169.10.254 |
| DNS | 8.8.8.8 |

**Una cosa sobre el fitxer de concessions.**

L'enunciat diu `/var/lib/kea/dhcp4.leases`.

Jo faig servir `/var/lib/kea/kea-leases4.csv`.

És el que crea el Kea tot sol. Dins hi ha el mateix. Només canvia el nom.

Ho explico més avall, als problemes.

---

## 5. Engegar el