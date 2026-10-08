# Fitxa tècnica: Servidor DHCP amb Kea sobre Ubuntu Server

## Objectiu

Muntar un servidor DHCP amb Kea en un Ubuntu Server. Després comprovar amb un client Zorin que rep bé la IP, la porta d'enllaç i el DNS.

També he de gravar la negociació amb el Wireshark i fer una reserva d'IP per MAC.

El meu número de llista és el 10. Per això faig servir la xarxa `192.169.10.0/24`.

## Materials

- Un ordinador amb Windows i VirtualBox.
- Una màquina virtual amb Ubuntu 24.04 Server. És el servidor.
- Una màquina virtual amb Zorin OS. És el client.
- Connexió a Internet per baixar els paquets.

## Com està muntat

| Màquina | Rol | Sistema |
|---|---|---|
| `srv-smx01` | Servidor DHCP | Ubuntu 24.04 Server |
| `cliente zorin pratica` | Client | Zorin OS |

El servidor té dues targetes de xarxa:

| Targeta | Tipus | Adreça |
|---|---|---|
| `enp0s3` | NAT | 10.0.2.15 |
| `enp0s8` | Xarxa interna `SMX-LAB` | 192.169.10.1 |

La primera serveix per sortir a Internet. La segona és la que fa servir el DHCP.

A la segona no hi poso porta d'enllaç ni DNS. Ho diu l'enunciat.

El client només té una targeta. Primer la poso en NAT. Després la canvio a la xarxa interna.

---

## Pas 1. Configurar la xarxa del servidor

Obro el fitxer de netplan.

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

Ho guardo i ho aplico.

```bash
sudo netplan apply
ip a
```

Amb `ip a` miro que `enp0s8` tingui la IP `192.169.10.1`.

---

## Pas 2. Instal·lar Kea

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install kea
```

Mentre s'instal·la surt una pantalla blava. Em demana una contrasenya.

Trio l'opció `configured_random_password`. Jo treballo directament al servidor, no des de fora. Per això aquesta contrasenya no m'importa.

Si m'equivoco puc tornar a veure la pantalla amb `sudo dpkg-reconfigure kea-ctrl-agent`.

El Kea instal·la quatre serveis. Van per separat.

| Servei | El seu fitxer | Per a què serveix |
|---|---|---|
| `kea-dhcp4-server` | `/etc/kea/kea-dhcp4.conf` | DHCP per a IPv4 |
| `kea-dhcp6-server` | `/etc/kea/kea-dhcp6.conf` | DHCP per a IPv6 |
| `kea-dhcp-ddns-server` | `/etc/kea/kea-dhcp-ddns.conf` | DDNS |
| `kea-ctrl-agent` | `/etc/kea/kea-ctrl-agent.conf` | Control remot |

---

## Pas 3. Apagar el DHCPv6 i el DDNS

Jo només vull el DHCP per a IPv4. Per tant apago els altres dos.

```bash
sudo systemctl stop kea-dhcp6-server
sudo systemctl disable kea-dhcp6-server

sudo systemctl stop kea-dhcp-ddns-server
sudo systemctl disable kea-dhcp-ddns-server
```

Compte amb el nom del DDNS. Es diu `kea-dhcp-ddns-server`. A classe surt escrit `kea-dhcp-ddns` i amb aquest nom no funciona.

Ara ho comprovo.

```bash
systemctl list-unit-files | grep kea
```

El dhcp6 i el ddns han de sortir com a `disabled`. El dhcp4 ha de sortir com a `enabled`.

![Els serveis del Kea](/img/1.png)

---

## Pas 4. Configurar el fitxer kea-dhcp4.conf

Primer canvio el nom del fitxer que ve de sèrie. Així no el perdo i el puc mirar si vull veure exemples.

```bash
cd /etc/kea
sudo mv kea-dhcp4.conf old-kea-dhcp4.conf
sudo nano kea-dhcp4.conf
```

El fitxer és en format JSON. Els blocs van entre `{ }` i les llistes entre `[ ]`. He de mirar molt bé on va coma i on no.

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

Una cosa sobre el fitxer de concessions. L'enunciat diu `/var/lib/kea/dhcp4.leases`. Jo faig servir `/var/lib/kea/kea-leases4.csv`, que és el que crea el Kea tot sol i ja té els permisos bons. Dins hi ha el mateix, només canvia el nom.

---

## Pas 5. Comprovar i arrencar el servei

Abans de res miro si el fitxer està ben escrit.

```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

No ha de sortir cap línia que posi `ERROR`. Les que posen `WARN` són normals.

Ara l'engego.

```bash
sudo systemctl restart kea-dhcp4-server
sudo systemctl status kea-dhcp4-server
```

Ha de sortir `active (running)` en verd.

![El servei funcionant](/img/2.png)

---

## Pas 6. Preparar el client

El client l'engego en NAT. Necessita Internet per baixar el Wireshark.

```bash
sudo apt update
sudo apt install wireshark
sudo wireshark
```

![Instal·lant el Wireshark](/img/3.png)

![El Wireshark obert](/img/4.png)

---

## Pas 7. Capturar la negociació

Aquí l'ordre és molt important. Si canvio la xarxa abans de començar a gravar, perdo els paquets.

**1.** Començo a gravar a la targeta `enp0s3`. A dalt poso el filtre `dhcp`.

**2.** Sense aturar la gravació, canvio el client de NAT a xarxa interna `SMX-LAB`.

![El client a la xarxa interna](/img/5.png)

**3.** Apago i torno a encendre la connexió cablejada des de la icona de xarxa. També es pot fer escrivint això:

```bash
sudo nmcli device down enp0s3 && sudo nmcli device up enp0s3
```

**4.** Aturo la gravació i la deso.

![Els paquets DHCP](/img/6.png)

Hi surten els quatre paquets: Discover, Offer, Request i ACK. També hi surten dos NAK.

---

## Pas 8. Quins paquets són broadcast i quins unicast

Per saber-ho he obert cada paquet. He mirat el bloc `Ethernet II`, on surten les MAC, i el bloc `Internet Protocol`, on surten les IP.

| Paquet | IP origen → destí | MAC destí | Tipus |
|---|---|---|---|
| Discover | 0.0.0.0 → 255.255.255.255 | `ff:ff:ff:ff:ff:ff` | Broadcast IP i MAC |
| Offer | 192.169.10.1 → 192.169.10.55 | `08:00:27:45:64:24` | Unicast IP i MAC |
| Request | 0.0.0.0 → 255.255.255.255 | `ff:ff:ff:ff:ff:ff` | Broadcast IP i MAC |
| ACK | 192.169.10.1 → 192.169.10.55 | `08:00:27:45:64:24` | Unicast IP i MAC |
| NAK | 192.169.10.1 → 255.255.255.255 | `08:00:27:45:64:24` | Broadcast IP, unicast MAC |

**El Discover és broadcast**, a nivell IP i a nivell MAC. El client encara no té cap IP ni sap quins servidors DHCP hi ha. Per això ho envia a tothom. Com a origen fa servir 0.0.0.0, que és el que es posa quan encara no tens adreça.

**L'Offer és unicast.** El servidor ja sap la MAC del client, perquè li venia dins del Discover. A més, el camp `Bootp flags` està a 0. Per això li pot contestar directament sense molestar tota la xarxa.

**El Request és broadcast.** El client encara no té l'adreça confirmada. I enviant-ho a tothom avisa els altres servidors DHCP que ja ha triat una oferta, perquè alliberin les adreces que tenien guardades.

**L'ACK és unicast.** Aquí ja està tot confirmat. El servidor sap quina IP ha donat i sap la MAC del client.

**El NAK és un cas especial.** És broadcast a nivell IP però unicast a nivell MAC. Surt perquè el client volia renovar la IP que tenia quan estava en NAT. Aquella IP no és de la xarxa 192.169.10.0/24, així que el servidor la rebutja i l'obliga a tornar a començar amb un Discover nou. La IP de destí és broadcast perquè el client no té cap IP bona en aquesta xarxa, però el servidor sí que sap la seva MAC i li pot enviar la trama directament.

**Resum.** Tot el que envia el client és broadcast, perquè encara no té IP ni sap qui és el servidor. Tot el que envia el servidor és unicast, perquè ja sap la MAC del client.

---

## Pas 9. Comprovar el client

Al Zorin vaig a `Configuración → Red`. Clico la roda dentada de la connexió cablejada i entro a `Detalles`.

| Camp | Valor |
|---|---|
| Adreça IPv4 | 192.169.10.10 |
| Ruta predeterminada | 192.169.10.254 |
| DNS | 8.8.8.8 |
| Adreça física | 08:00:27:45:64:24 |

La IP és del pool. La porta d'enllaç i el DNS són els que vaig posar al fitxer. Funciona.

![El client amb la IP del pool](/img/7.png)

Al servidor miro les concessions.

```bash
cat /var/lib/kea/kea-leases4.csv
```

Cada cop que el client renova, s'hi afegeix una línia nova. L'última és la que val.

![El fitxer de concessions](/img/8.png)

---

## Pas 10. Fer la reserva

Vull que el client tingui sempre la 192.169.10.55. Per fer-ho he de dir la seva adreça MAC.

La reserva va dins del bloc de la subxarxa, però fora del pool. Si estigués dins del pool, el servidor podria donar aquella IP a un altre ordinador i hi hauria conflicte.

```bash
sudo nano /etc/kea/kea-dhcp4.conf
```

```json
        "pools": [ { "pool": "192.169.10.10 - 192.169.10.50" } ],
        "reservations": [
          {
            "hw-address": "08:00:27:45:64:24",
            "ip-address": "192.169.10.55"
          }
        ]
```

Dues coses que he de vigilar:

- Després del `]` del pool ara hi va una coma. Ja no és l'últim del bloc.
- La MAC ha de ser la del meu client. No la de l'exemple de classe.

![El fitxer amb la reserva](/img/9.png)

Ara ho aplico.

```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
sudo systemctl restart kea-dhcp4-server
sudo systemctl status kea-dhcp4-server
```

![El servei després de la reserva](/img/10.png)

Al client apago i torno a encendre la connexió. Ara ja em dóna la 192.169.10.55.

![El client amb la IP reservada](/img/11.png)

Al fitxer de concessions es veu tot el canvi. Primer les línies amb la 192.169.10.10. Després una línia amb el `valid_lifetime` a 0, que vol dir que allibera aquella adreça. I al final les línies noves amb la 192.169.10.55.

![Les concessions amb la reserva](/img/12.png)

---

## Problemes que he tingut

| Què em sortia | Per què | Com ho he arreglat |
|---|---|---|
| `Unit kea-dhcp-ddns.service not loaded` | El servei es diu `kea-dhcp-ddns-server`, no `kea-dhcp-ddns` | Posar el nom sencer |
| `Failed to parse pool definition: 192.169.x.10` | Havia copiat l'exemple de classe i m'havia deixat la `x` | Posar-hi el meu número de llista |
| `Unable to open database: unable to open '/var/lib/kea/dhcp4.leases'` | El Kea funciona com a usuari `_kea` i el fitxer que jo havia fet era de `root` | Fer servir `kea-leases4.csv`, que ja té els permisos bons |
| `enp0s8` tenia dues IP alhora | El netplan afegeix adreces noves però no esborra les velles | `sudo reboot` |
| El client seguia agafant la IP del pool | Havia copiat la MAC de l'exemple de classe | Posar-hi la MAC del meu client |

---

## Conclusions

El Kea substitueix l'antic `isc-dhcp-server`, que ja no té suport. El canvi més gros és que ara la configuració és en JSON i que els serveis es gestionen amb systemd, no amb l'eina `keactrl`.

El DHCP en si no m'ha costat d'entendre. El que m'ha costat han estat els detalls petits: el nom exacte dels serveis, els permisos dels fitxers i sobretot haver copiat la MAC de l'exemple. Per culpa d'això la reserva no anava i jo no entenia per què.

## Documentació

- [KEA — The DHCPv4 Server](https://kea.readthedocs.io/en/kea-1.6.2/arm/dhcp4-srv.html)
- [Exemple de configuració](https://github.com/carlesalonso/kea-dhcp4-demo)