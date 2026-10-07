# Fitxa tècnica: Servidor DHCP amb Kea

## Objectiu

Muntar un servidor DHCP amb Kea en un Ubuntu Server. Després comprovar amb un client Zorin que rep bé la IP, la porta d'enllaç i el DNS.

També he de capturar la negociació amb Wireshark i fer una reserva d'IP per MAC.

El meu número de llista és el 10. Per tant faig servir la xarxa `192.169.10.0/24`.

## Materials

- Un ordinador amb Windows i VirtualBox.
- Una màquina virtual amb Ubuntu 24.04 Server. És el servidor.
- Una màquina virtual amb Zorin OS. És el client.
- Connexió a Internet per baixar els paquets.

## Com està muntat

El servidor té tres adaptadors de xarxa:

| Adaptador | Tipus | Interfície | Adreça |
|---|---|---|---|
| 1 | NAT | `enp0s3` | 10.0.2.15 |
| 2 | Xarxa interna `SMX-LAB` | `enp0s8` | 192.169.10.1 |
| 3 | Només l'amfitrió | `enp0s9` | 192.168.56.x |

El primer serveix per sortir a Internet. El segon és el que fa servir el DHCP. El tercer no el demana l'enunciat. L'he posat per entrar per SSH des de Windows i no haver de treballar dins de la finestra de VirtualBox.

El client només té un adaptador. Primer el poso en NAT. Més endavant el canvio a la xarxa interna.

![Adaptadors del servidor](img/01-adaptadors-servidor.png)

![El client a la xarxa interna](img/02-client-xarxa-interna.png)

---

## Pas 1. Configurar la xarxa del servidor

Primer miro com es diuen les interfícies. Poden canviar de nom.

```bash
ip a
```

Obro el fitxer de netplan.

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

I hi poso això:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      addresses: [192.169.10.1/24]
    enp0s9:
      dhcp4: true
```

A `enp0s8` no hi poso porta d'enllaç ni DNS. La sortida a Internet va per `enp0s3`.

Ara ho aplico i ho comprovo.

```bash
sudo chmod 600 /etc/netplan/00-installer-config.yaml
sudo netplan apply
ip a
```

![El fitxer netplan](img/03-netplan.png)

![Comprovació amb ip a](img/04-ip-a.png)

---

## Pas 2. Instal·lar Kea

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install kea
```

Durant la instal·lació surt una pantalla blava. Em demana una contrasenya per a l'API.

Trio l'opció `configured_random_password`. Jo treballo directament al servidor, no de forma remota. Per tant aquesta contrasenya no m'afecta.

Si m'equivoco puc tornar a fer la pregunta amb `sudo dpkg-reconfigure kea-ctrl-agent`.

Kea instal·la quatre serveis. Són independents entre ells.

| Unitat systemd | Fitxer de configuració | Per a què serveix |
|---|---|---|
| `kea-dhcp4-server` | `/etc/kea/kea-dhcp4.conf` | DHCP per a IPv4 |
| `kea-dhcp6-server` | `/etc/kea/kea-dhcp6.conf` | DHCP per a IPv6 |
| `kea-dhcp-ddns-server` | `/etc/kea/kea-dhcp-ddns.conf` | DDNS |
| `kea-ctrl-agent` | `/etc/kea/kea-ctrl-agent.conf` | Control remot |

---

## Pas 3. Apagar el DHCPv6 i el DDNS

Només em fa falta el DHCPv4. Els altres dos els paro i els deshabilito.

```bash
sudo systemctl stop kea-dhcp6-server
sudo systemctl disable kea-dhcp6-server

sudo systemctl stop kea-dhcp-ddns-server
sudo systemctl disable kea-dhcp-ddns-server
```

Compte amb el nom del DDNS. Es diu `kea-dhcp-ddns-server`. Si poso `kea-dhcp-ddns` em diu que la unitat no existeix.

Ho comprovo:

```bash
systemctl list-unit-files | grep kea
```

El dhcp6 i el ddns han de sortir com a `disabled`. El dhcp4 ha de sortir com a `enabled`.

![Els serveis de Kea](img/06-serveis-kea.png)

---

## Pas 4. Configurar el fitxer kea-dhcp4.conf

Canvio el nom del fitxer original. Així no el perdo i el puc mirar si vull veure exemples.

```bash
cd /etc/kea
sudo mv kea-dhcp4.conf old-kea-dhcp4.conf
sudo nano kea-dhcp4.conf
```

El fitxer és en format JSON. Els blocs s'obren i es tanquen amb `{ }`. Els subapartats van entre `[ ]`. Cal mirar bé on va coma i on no.

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
| Pool | 192.169.10.10 fins a 192.169.10.50 |
| Porta d'enllaç | 192.169.10.254 |
| DNS | 8.8.8.8 |

![El fitxer kea-dhcp4.conf](img/07-kea-dhcp4-conf.png)

Una cosa important. L'enunciat parla del fitxer `/var/lib/kea/dhcp4.leases`. Jo faig servir `/var/lib/kea/kea-leases4.csv`. És el que crea el paquet sol