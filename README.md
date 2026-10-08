# Fitxa tècnica: Servidor DHCP amb Kea sobre Ubuntu Server

## Objectiu

Muntar un servidor DHCP amb Kea en un Ubuntu Server. Després comprovar amb un client Zorin que rep bé la IP, la porta d'enllaç i el DNS.

També cal capturar la negociació amb Wireshark i fer una reserva d'IP per MAC.

El meu número de llista és el 10. Per tant faig servir la xarxa `192.169.10.0/24`.

## Escenari

| Màquina | Rol | Sistema |
|---|---|---|
| `srv-smx01` | Servidor DHCP | Ubuntu 24.04 Server |
| `cliente zorin pratica` | Client | Zorin OS |

El servidor té dues interfícies:

- `enp0s3` en **NAT**. Serveix per sortir a Internet.
- `enp0s8` en **xarxa interna `SMX-LAB`**, amb l'adreça `192.169.10.1/24`. És la que fa servir el DHCP.

A la segona interfície no hi poso ni porta d'enllaç ni servidor de noms, tal com demana l'enunciat.

El client només té un adaptador. Primer el poso en NAT per instal·lar el Wireshark. Després el canvio a la xarxa interna.

---

## Pas 1. Configurar la xarxa del servidor

Obro el fitxer de netplan.

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
```

Ho aplico i ho comprovo.

```bash
sudo netplan apply
ip a
```

---

## Pas 2. Instal·lar Kea

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install kea
```

Durant la instal·lació surt una pantalla blava. Em demana una contrasenya per a l'API.

Trio l'opció `configured_random_password`. Com que treballo directament al servidor i no de forma remota, aquesta contrasenya no m'afecta.

Si m'equivoco puc tornar-ho a fer amb `sudo dpkg-reconfigure kea-ctrl-agent`.

Kea instal·la quatre serveis independents:

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

Compte amb el nom del DDNS. Es diu `kea-dhcp-ddns-server`. Si poso `kea-dhcp-ddns` em surt un error dient que la unitat no existeix.

Ho comprovo:

```bash
systemctl list-unit-files | grep kea
```

El dhcp6 i el ddns surten com a `disabled`. El dhcp4 surt com a `enabled`.

![Els serveis de Kea](img/01-serveis-kea.png)

---

## Pas 4. Configurar el fitxer kea-dhcp4.conf

Canvio el nom del fitxer original per no perdre'l. Així el puc mirar si vull veure exemples.

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

Una cosa important. L'enunciat parla del fitxer `/var/lib/kea/dhcp4.leases`. Jo faig servir `/var/lib/kea/kea-leases4.csv`. És el que crea el paquet sol i amb els permisos bons. El contingut és el mateix. Només canvia el nom que poso a `lease-database`.

---

## Pas 5. Comprovar i arrencar el servei

Primer miro si el fitxer està ben escrit.

```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

No ha de sortir cap línia que posi `ERROR`. Les que posen `WARN` són normals.

Ara reinicio i miro com està.

```bash
sudo systemctl restart kea-dhcp4-server
sudo systemctl status kea-dhcp4-server
```

Ha de posar `active (running)` en verd.

![El servei funcionant](img/02-status-active.png)

---

## Pas 6. Preparar el client

El client l'arrenco en NAT. Necessita Internet per baixar el Wireshark.

```bash
sudo apt update
sudo apt install wireshark
sudo wireshark
```

![Instal·lant el Wireshark](img/03-instal-wireshark.png)

![El Wireshark obert](img/04-wireshark-obert.png)

---

## Pas 7. Capturar la negociació

Aquí l'ordre és molt important. Si canvio la xarxa abans de començar a gravar, perdo els paquets.

**1.** Començo a capturar a la interfície `enp0s3`. Poso `dhcp` al filtre de dalt.

**2.** Sense aturar la captura, canvio el client de NAT a xarxa interna `SMX-LAB`.

![El client a la xarxa interna](img/05-client-xarxa-interna.png)

**3.** Desactivo i torno a activar la connexió cablejada des de l'eina gràfica. També es pot fer per terminal:

```bash
sudo nmcli device down enp0s3 && sudo nmcli device up enp0s3
```

**4.** Aturo la captura i la deso en format `.pcapng`.

![Els paquets DHCP capturats](img/06-paquets-dhcp.png)

A la captura es veuen els quatre paquets de la negociació: Discover, Offer, Request i ACK. També hi surten dos paquets NAK.

---

## Pas 8. Quins paquets són broadcast i quins unicast

Per saber-ho obro cada paquet i miro el bloc `Ethernet II`, on surten les MAC, i el bloc `Internet Protocol`, on surten les IP.

| Paquet | IP origen → destí | MAC destí | Tipus |
|---|---|---|---|
| Discover | 0.0.0.0 → 255.255.255.255 | `ff:ff:ff:ff:ff:ff` | Broadcast IP i MAC |
| Offer | 192.169.10.1 → 192.169.10.55 | `08:00:27:45:64:24` | Unicast IP i MAC |
| Request | 0.0.0.0 → 255.255.255.255 | `ff:ff:ff:ff:ff:ff` | Broadcast IP i MAC |
| ACK | 192.169.10.1 → 192.169.10.55 | `08:00:27:45:64:24` | Unicast IP i MAC |
| NAK | 192.169.10.1 → 255.255.255.255 | `08:00:27:45:64:24` | Broadcast IP i unicast MAC |

**El Discover és broadcast.** Ho és a nivell IP i a nivell MAC. El client encara no té cap IP. Tampoc sap quins servidors DHCP hi ha a la xarxa. Per això ho envia a tothom. Com a origen fa servir 0.0.0.0, que és el que es posa quan encara no tens adreça.

**L'Offer és unicast.** El servidor ja sap la MAC del client, perquè venia dins del Discover. A més, el bit de broadcast del camp `Bootp flags` està a 0. Per tant pot contestar directament sense molestar la resta de la xarxa.

**El Request és broadcast.** El client encara no té l'adreça confirmada. A més, enviant-ho a tothom avisa els altres servidors DHCP que ja ha triat una oferta. Així poden alliberar les adreces que tenien guardades.

**L'ACK és unicast.** Ja està tot confirmat. El servidor sap la IP que ha donat i sap la MAC del client.

**El NAK és un cas especial.** És broadcast a nivell IP però unicast a nivell MAC. Surt perquè el client intentava renovar la IP que tenia quan estava en NAT. Aquella IP no és de la xarxa 192.169.10.0/24, així que el servidor la rebutja i l'obliga a tornar a començar amb un Discover nou. La IP de destí és broadcast perquè el client no té cap IP bona en aquesta xarxa. Però el servidor sí que sap la seva MAC i li pot enviar la trama directament.

**Resum.** Tot el que envia el client és broadcast, perquè encara no té IP ni sap qui és el servidor. Tot el que envia el servidor és unicast, perquè ja sap la MAC del client.

---

## Pas 9. Comprovar el client

Al Zorin vaig a `Configuración → Red`. Clico la roda dentada de la connexió cablejada i entro a la pestanya `Detalles`.

| Camp | Valor |
|---|---|
| Adreça IPv4 | 192.169.10.10 |
| Ruta predeterminada | 192.169.10.254 |
| DNS | 8.8.8.8 |
| Adreça física | 08:00:27:45:64:24 |

La IP és del pool. La porta d'enllaç i el DNS són els que vaig posar al fitxer. Funciona.

![El client amb la IP del pool](img/07-client-ip-pool.png)

Al servidor miro les concessions.

```bash
cat /var/lib/kea/kea-leases4.csv
```

Cada vegada que el client renova, s'hi afegeix una línia nova. L'última és la que val.

![El fitxer de concessions](img/08-leases-pool.png)

---

## Pas 10. Fer la reserva

Vull que el client tingui sempre la 192.169.10.55. Per fer-ho he d'indicar la seva adreça MAC.

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

Dues coses a vigilar:

- Després del `]` del pool ara hi va una **coma**. Ja no és l'últim element del bloc.
- La MAC ha de ser la del meu client. No la de l'exemple dels apunts.

![El fitxer amb la reserva](img/09-conf-reserva.png)

Ara ho aplico.

```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
sudo systemctl restart kea-dhcp4-server
sudo systemctl status kea-dhcp4-server
```

![El servei després de la reserva](img/10-status-reserva.png)

Al client desactivo i torno a activar la connexió. Ara ja em dóna la 192.169.10.55.

![El client amb la IP reservada](img/11-client-ip-reservada.png)

Al fitxer de concessions es veu tot el canvi. Primer les línies amb la 192.169.10.10. Després una línia amb el `valid_lifetime` a 0, que vol dir que allibera aquella adreça. I al final les línies noves amb la 192.169.10.55.

![Les concessions amb la reserva](img/12-leases-reserva.png)

---

## Problemes que he tingut

| Què em sortia | Per què | Com ho he arreglat |
|---|---|---|
| `Unit kea-dhcp-ddns.service not loaded` | La unitat es diu `kea-dhcp-ddns-server` | Posar el nom sencer |
| `Failed to parse pool definition: 192.169.x.10` | Havia deixat la `x` de l'enunciat dins del fitxer | Posar-hi el meu número de llista |
| `Unable to open database: unable to open '/var/lib/kea/dhcp4.leases'` | Kea funciona com a usuari `_kea` i el fitxer era de `root` | Fer servir `kea-leases4.csv`, que ja té els permisos bons |
| `enp0s8` tenia dues IP alhora | Netplan afegeix adreces però no esborra les velles fins que reinicies | `sudo reboot` |
| El client no agafava la IP reservada | Havia copiat la MAC de l'exemple dels apunts | Posar-hi la MAC del meu client |

---

## Conclusions

Kea substitueix l'antic `isc-dhcp-server`, que ja no té suport dels seus creadors. El canvi més gros és que ara la configuració és en JSON i que els serveis es gestionen amb systemd, no amb l'eina `keactrl`.

El que més m'ha costat no ha estat entendre el DHCP. Han estat els detalls petits: el nom exacte de les unitats de systemd, els permisos del fitxer de concessions i sobretot haver copiat la MAC de l'exemple dels apunts. Per culpa d'això la reserva no s'aplicava mai i no entenia per què el client seguia agafant la IP del pool.

## Documentació

- [Ubuntu Server Docs — isc-kea](https://ubuntu.com/server/docs/how-to-install-and-configure-isc-kea)
- [KEA — The DHCPv4 Server](https://kea.readthedocs.io/en/kea-1.6.2/arm/dhcp4-srv.html)
- [Exemple de configuració](https://github.com/carlesalonso/kea-dhcp4-demo)