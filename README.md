# Fitxa tècnica: Servidor DHCP amb Kea sobre Ubuntu Server

## Objectiu

Instal·lar i configurar un servidor DHCP amb **Kea** sobre Ubuntu 24.04 Server, dins d'una xarxa interna de VirtualBox, i comprovar amb un client Zorin Linux que rep correctament la configuració de xarxa. També s'analitza la negociació DHCP amb Wireshark i es configura una reserva d'adreça per MAC.

El meu número de llista és el **10**, així que tota la pràctica es fa sobre la xarxa `192.169.10.0/24`.

## Materials

- Un ordinador amb Windows i VirtualBox.
- Una màquina virtual amb **Ubuntu 24.04 Server** (`srv-smx01`), que farà de servidor.
- Una màquina virtual amb **Zorin OS**, que farà de client.
- Connexió a Internet per instal·lar els paquets.

## Escenari

| Màquina | Rol | Sistema |
|---|---|---|
| `srv-smx01` | Servidor DHCP | Ubuntu 24.04 Server |
| `cliente zorin pratica` | Client | Zorin OS (Ubuntu 64-bit) |

El servidor té tres adaptadors de xarxa:

| Adaptador | Tipus | Interfície | Adreça |
|---|---|---|---|
| 1 | NAT | `enp0s3` | 10.0.2.15 — sortida a Internet |
| 2 | Xarxa interna `SMX-LAB` | `enp0s8` | 192.169.10.1/24 — servei DHCP |
| 3 | Adaptador de només l'amfitrió | `enp0s9` | 192.168.56.x — accés per SSH |

> El tercer adaptador no el demana l'enunciat. L'he afegit per poder administrar el servidor per SSH des de Windows en comptes de treballar dins la consola de VirtualBox.

El client només té un adaptador, que al principi està en NAT i després es canvia a la xarxa interna `SMX-LAB`.

![Adaptadors del servidor](img/01-adaptadors-servidor.png)

![Client a la xarxa interna SMX-LAB](img/02-client-xarxa-interna.png)

---

## Procediment

### 1. Configurar la xarxa del servidor

Primer miro com es diuen realment les interfícies, perquè poden canviar:

```bash
ip a
```

Edito el fitxer de netplan:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

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

A `enp0s8` no hi poso ni porta d'enllaç ni servidor de noms, perquè la sortida a Internet es fa per `enp0s3`.

```bash
sudo chmod 600 /etc/netplan/00-installer-config.yaml
sudo netplan apply
ip a
```

![Fitxer netplan](img/03-netplan.png)

![Comprovació amb ip a](img/04-ip-a.png)

### 2. Instal·lar Kea

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install kea
```

Durant la instal·lació, el paquet `kea-ctrl-agent` demana configurar l'autenticació de l'API. Trio l'opció **`configured_random_password`**. Com que treballo directament sobre el servidor i no de forma remota, aquesta contrasenya no m'afecta.

Si m'equivoco, es pot repetir el diàleg amb `sudo dpkg-reconfigure kea-ctrl-agent`.

Kea instal·la quatre serveis independents controlats per systemd:

| Unitat systemd | Fitxer de configuració | Servei |
|---|---|---|
| `kea-dhcp4-server` | `/etc/kea/kea-dhcp4.conf` | Servidor DHCP IPv4 |
| `kea-dhcp6-server` | `/etc/kea/kea-dhcp6.conf` | Servidor DHCP IPv6 |
| `kea-dhcp-ddns-server` | `/etc/kea/kea-dhcp-ddns.conf` | Servidor DDNS |
| `kea-ctrl-agent` | `/etc/kea/kea-ctrl-agent.conf` | Agent de control remot |

### 3. Desactivar el DHCPv6 i el DDNS

Com que només m'interessa el DHCPv4, paro i deshabilito els altres dos:

```bash
sudo systemctl stop kea-dhcp6-server
sudo systemctl disable kea-dhcp6-server

sudo systemctl stop kea-dhcp-ddns-server
sudo systemctl disable kea-dhcp-ddns-server
```

> **Compte:** la unitat del DDNS es diu `kea-dhcp-ddns-server`, no `kea-dhcp-ddns`. Amb el nom curt systemd respon `Unit kea-dhcp-ddns.service not loaded`.

Comprovo que han quedat desactivats:

```bash
systemctl list-unit-files | grep kea
```

![Serveis de Kea](img/06-serveis-kea.png)

### 4. Configurar el fitxer kea-dhcp4.conf

Canvio el nom del fitxer original per no perdre'l i en creo un de nou:

```bash
cd /etc/kea
sudo mv kea-dhcp4.conf old-kea-dhcp4.conf
sudo nano kea-dhcp4.conf
```

El format és JSON, així que cal vigilar les claus `{ }`, els claudàtors `[ ]` i on va coma i on no.

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

Els paràmetres que demana l'enunciat:

| Paràmetre | Valor |
|---|---|
| Pool | 192.169.10.10 – 192.169.10.50 |
| Porta d'enllaç | 192.169.10.254 |
| DNS | 8.8.8.8 |

![Fitxer kea-dhcp4.conf](img/07-kea-dhcp4-conf.png)

> **Nota sobre el fitxer de concessions.** L'enunciat parla de `/var/lib/kea/dhcp4.leases`, però jo faig servir `/var/lib/kea/kea-leases4.csv`, que és el que crea el paquet per defecte i amb els permisos correctes per a l'usuari `_kea`. El contingut és el mateix, només canvia el nom definit a `lease-database`.

### 5. Validar i arrencar el servei

```bash
sudo kea-dhcp4 -t