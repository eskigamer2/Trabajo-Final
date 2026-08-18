# NetSecure Solutions — Sede Santiago

## Rol en la topología

Santiago es la **sede secundaria de mayor peso**: aquí vive el datacenter
corporativo (`DATACENTER-ST-001`, IP `10.1.0.34`) con los 5 servicios
centralizados de la empresa, y el switch de distribución (`SWMD-ST-1`) hace
**enrutamiento de Capa 3** directo con el router `R-ST` (a diferencia de las
demás sedes, donde el switch de distribución es solo Capa 2).

### Servicios del Datacenter (servidor único `10.1.0.34`)

| Servicio | Detalle |
|---|---|
| DNS | Dominio `netsecure.com.do` |
| DHCP | Asigna IP a las VLANs de Santiago |
| Web | `netsecure.com.do` (expuesto a Internet vía NAT estático en el [ISP](./ISP.md)) |
| RADIUS | Autenticación AAA de **todos** los dispositivos de la red (puerto 1812/1813) — usuario por cada integrante del grupo |
| Correo | Dominio `netsecure.com.do` — cuenta por cada integrante del grupo |

> **Gateway del servidor:** `10.1.0.33` (VLAN 220 — DATACENTER-ST)

### VLANs

| VLAN | Nombre | Subred |
|---:|---|---|
| 200 | DEFAULT-ST | — |
| 210 | VENTAS-ST | `10.1.0.0/27` |
| 220 | DATACENTER-ST | `10.1.0.32/27` |
| 230 | ADMIN-ST | `10.1.0.64/28` |
| 299 | MGMT-ST | `10.1.1.0/28` |

### Dispositivos

- `R-ST` — Router de sede (WAN al ISP + enlace L3 al switch de distribución + spoke DMVPN)
- `SWMD-ST-1` — Switch de distribución/núcleo, **con enrutamiento (`ip routing`)**, gateways SVI de cada VLAN
- `SWA-ST-1` — Switch de acceso (puertos de usuario, DHCP snooping, DAI)

---

## Configuraciones

<details>
<summary><strong>R-ST (Router)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname R-ST
no ip domain lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1

ip cef
no ipv6 cef

!
! ============================================================
! BANNER
! ============================================================
!

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (SANTIAGO)
ADVERTENCIA: ACCESO RESTRINGIDO. AUTENTICACION RADIUS ACTIVA.
*********************************************************************
#

!
! ============================================================
! SSH
! ============================================================
!

crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 2

line console 0
 logging synchronous
 exec-timeout 10 0
exit

line vty 0 4
 transport input ssh
 exec-timeout 10 0
exit

!
! ============================================================
! RADIUS / AAA
! ============================================================
!

aaa new-model
aaa authentication login default group radius local

radius server SANTIAGO-RADIUS
 address ipv4 10.1.0.34 auth-port 1812 acct-port 1813
 key Netsecure-EMP1
exit

!
! ============================================================
! INTERFACES
! ============================================================
!

interface Ethernet0/0
 description ENLACE-WAN-ISP
 ip address 1.0.0.6 255.255.255.252
 ip nat outside
 ip virtual-reassembly in
 duplex full
 no shutdown
exit

interface Ethernet0/1
 description ENLACE-SWMD-ST-1
 ip address 10.6.2.1 255.255.255.252
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf network point-to-point
 ip ospf 21 area 2
 duplex full
 no shutdown
exit

!
! ============================================================
! OSPF / ENRUTAMIENTO
! ============================================================
!

router ospf 21
 router-id 2.2.2.2
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface Ethernet0/1
 no passive-interface Tunnel1
exit

ip route 0.0.0.0 0.0.0.0 1.0.0.5

!
! ============================================================
! NAT
! ============================================================
!

ip access-list extended NAT-ACL-ST
 deny ip 10.1.0.0 0.0.255.255 10.0.0.0 0.255.255.255
 deny ip 10.1.0.0 0.0.255.255 1.0.0.24 0.0.0.7
 permit ip 10.1.0.0 0.0.255.255 any
exit

ip nat inside source list NAT-ACL-ST interface Ethernet0/0 overload

!
! ============================================================
! VPN / ISAKMP
! ============================================================
!

crypto isakmp policy 21
 encr aes
 hash sha512
 authentication pre-share
 group 14
exit

crypto isakmp key Cisco123 address 1.0.0.2

!
! ============================================================
! IPSEC
! ============================================================
!

crypto ipsec transform-set TS-VPN esp-aes esp-sha512-hmac
 mode transport
exit

crypto ipsec profile PF-VPN
 set transform-set TS-VPN
exit

!
! ============================================================
! DMVPN / NHRP
! ============================================================
!

interface Tunnel1
 description DMVPN-SPOKE-SANTIAGO
 ip address 10.7.0.2 255.255.255.240
 no ip redirects
 ip mtu 1400
 ip nhrp authentication NETSECURE
 ip nhrp map 10.7.0.1 1.0.0.2
 ip nhrp map multicast 1.0.0.2
 ip nhrp network-id 2026
 ip nhrp nhs 10.7.0.1
 ip tcp adjust-mss 1360
 ip ospf network point-to-multipoint
 ip ospf 21 area 0
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile PF-VPN
 no shutdown
exit

!
! ============================================================
! SEGURIDAD
! ============================================================
!

no ip http server
no ip http secure-server

end
write memory
```

</details>

<details>
<summary><strong>SWMD-ST-1 (Switch de Distribución / Núcleo)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname SWMD-ST-1
no ip domain-lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain-name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1

ip routing
ip cef
no ipv6 cef

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (SANTIAGO)
ADVERTENCIA: ACCESO RESTRINGIDO. AUTENTICACION RADIUS ACTIVA.
*********************************************************************
#

crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 2

line console 0
 logging synchronous
 exec-timeout 10 0
exit

line vty 0 4
 transport input ssh
 exec-timeout 10 0
exit

aaa new-model
aaa authentication login default group radius local

radius server SANTIAGO-RADIUS
 address ipv4 10.1.0.34 auth-port 1812 acct-port 1813
 key Netsecure-EMP1
exit

!
! ============================================================
! VLANS
! ============================================================
!

vlan 200
 name DEFAULT-ST
exit

vlan 210
 name VENTAS-ST
exit

vlan 220
 name DATACENTER-ST
exit

vlan 230
 name ADMIN-ST
exit

vlan 299
 name MGMT-ST
exit

!
! ============================================================
! SPANNING TREE
! ============================================================
!

spanning-tree mode pvst
spanning-tree vlan 200,210,220,230,299 priority 24576

!
! ============================================================
! ENLACE CAPA 3 HACIA R-ST
! ============================================================
!

interface Ethernet0/0
 description ENLACE-R-ST
 no switchport
 ip address 10.6.2.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf 21 area 2
 duplex full
 no shutdown
exit

!
! ============================================================
! ETHERCHANNEL / TRUNK HACIA SWA-ST-1
! ============================================================
!

interface range Ethernet0/1-3
 description ETHERCHANNEL-L2-SWA-ST-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 200,210,220,230,299
 switchport trunk native vlan 200
 switchport nonegotiate
 duplex full
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 description TRUNK-A-SWA-ST-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 200,210,220,230,299
 switchport trunk native vlan 200
 switchport nonegotiate
 no shutdown
exit

!
! ============================================================
! GATEWAY VLAN 210 - VENTAS
! ============================================================
!

interface Vlan210
 description GATEWAY-VENTAS
 ip address 10.1.0.1 255.255.255.224
 ip helper-address 10.1.0.34
 ip ospf 21 area 2
 no shutdown
exit

!
! ============================================================
! GATEWAY VLAN 220 - DATACENTER
! ============================================================
!

interface Vlan220
 description GATEWAY-DATACENTER
 ip address 10.1.0.33 255.255.255.224
 ip ospf 21 area 2
 no shutdown
exit

!
! ============================================================
! GATEWAY VLAN 230 - ADMINISTRACION
! ============================================================
!

interface Vlan230
 description GATEWAY-ADMIN
 ip address 10.1.0.65 255.255.255.240
 ip helper-address 10.1.0.34
 ip ospf 21 area 2
 no shutdown
exit

!
! ============================================================
! MANAGEMENT - VLAN 299
! ============================================================
!

interface Vlan299
 description GATEWAY-MGMT
 ip address 10.1.1.1 255.255.255.240
 ip helper-address 10.1.0.34
 ip ospf 21 area 2
 no shutdown
exit

ip radius source-interface Vlan299

!
! ============================================================
! OSPF
! ============================================================
!

router ospf 21
 router-id 2.2.2.11
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface Ethernet0/0
exit

!
! ============================================================
! SEGURIDAD
! ============================================================
!

no ip http server
no ip http secure-server

end
write memory
```

</details>

<details>
<summary><strong>SWA-ST-1 (Switch de Acceso)</strong></summary>

```text
enable
configure terminal

!
! ============================================================
! CONFIGURACION INICIAL
! ============================================================
!

hostname SWA-ST-1
no ip domain-lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain-name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (SANTIAGO)
ADVERTENCIA: ACCESO RESTRINGIDO. AUTENTICACION RADIUS ACTIVA.
*********************************************************************
#

crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 2

line console 0
 logging synchronous
 exec-timeout 10 0
exit

line vty 0 4
 transport input ssh
 exec-timeout 10 0
exit

aaa new-model
aaa authentication login default group radius local

radius server SANTIAGO-RADIUS
 address ipv4 10.1.0.34 auth-port 1812 acct-port 1813
 key Netsecure-EMP1
exit

no ip routing

vlan 200
 name DEFAULT-ST
exit

vlan 210
 name VENTAS-ST
exit

vlan 220
 name DATACENTER-ST
exit

vlan 230
 name ADMIN-ST
exit

vlan 299
 name MGMT-ST
exit

spanning-tree mode pvst

!
! ============================================================
! ETHERCHANNEL / TRUNK
! ============================================================
!

interface range Ethernet0/1-3
 description ETHERCHANNEL-L2-SWMD-ST-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 200,210,220,230,299
 switchport trunk native vlan 200
 switchport nonegotiate
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 description TRUNK-A-SWMD-ST-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 200,210,220,230,299
 switchport trunk native vlan 200
 switchport nonegotiate
 no shutdown
exit

!
! ============================================================
! PUERTO MANAGEMENT
! ============================================================
!

interface Ethernet0/0
 description MGMT-ST
 switchport mode access
 switchport access vlan 299
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

!
! ============================================================
! VENTAS - VLAN 210
! ============================================================
!

interface range Ethernet1/0-3
 description VENTAS-ST
 switchport mode access
 switchport access vlan 210
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

!
! ============================================================
! DATACENTER - VLAN 220
! ============================================================
!

interface range Ethernet2/0-3
 description DATACENTER-ST
 switchport mode access
 switchport access vlan 220
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

!
! ============================================================
! ADMINISTRACION - VLAN 230
! ============================================================
!

interface range Ethernet3/0-3
 description ADMIN-ST
 switchport mode access
 switchport access vlan 230
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

!
! ============================================================
! MANAGEMENT SVI
! ============================================================
!

interface Vlan299
 description MANAGEMENT-SWA-ST-1
 ip address 10.1.1.3 255.255.255.240
 no shutdown
exit

ip default-gateway 10.1.1.1
ip radius source-interface Vlan299

no ip http server
no ip http secure-server

!
! ============================================================
! DHCP SNOOPING
! ============================================================
!

ip dhcp snooping vlan 210,230
no ip dhcp snooping information option
ip dhcp snooping

interface Port-channel1
 ip dhcp snooping trust
exit

interface range Ethernet0/1-3
 ip dhcp snooping trust
exit

interface range Ethernet1/0-3
 ip dhcp snooping limit rate 15
exit

interface range Ethernet3/0-3
 ip dhcp snooping limit rate 15
exit

!
! ============================================================
! DAI / ARP INSPECTION
! ============================================================
!

arp access-list DATACENTER-STATIC
 permit ip host 10.1.0.34 mac host 50a7.bb00.9000
exit

ip arp inspection filter DATACENTER-STATIC vlan 220
ip arp inspection vlan 210,220,230

interface Port-channel1
 ip arp inspection trust
exit

interface range Ethernet0/1-3
 ip arp inspection trust
exit

end
write memory
```

</details>
<details>
<summary><strong>Configuracion de DataCenter</strong></summary>

### 0. Disco de Servidores

https://drive.google.com/drive/folders/1KWONu9qGPgur1btCuTguQ5ALZNX8yjlU?usp=sharing

### 1. Configuración de Red (IP Estática)
```text
El servidor (`10.1.0.34`) reside en la **VLAN 220** (Gateway `.33`).

```bash
sudo nano /etc/netplan/01-network-manager-all.yaml

```

```yaml
network:
  version: 2
  ethernets:
    ens3:
      addresses: [10.1.0.34/27]
      routes:
        - to: default
          via: 10.1.0.33
      nameservers:
        addresses: [10.1.0.34, 8.8.8.8]

```

```bash
sudo netplan apply

```

---

### 2. Creación de Usuarios (Sistema, FTP y Correo)

Estos comandos crearán a los 6 miembros de la empresa con la estructura de contraseñas estandarizada (Ej: `Keury2026*`, `Isamar2026*`).

```bash
for u in isamar keury sahir lizbeth starling josue; do
  sudo adduser --disabled-password --gecos "" $u
  echo "$u:${u^}2026*" | sudo chpasswd
done

```

---

### 3. Servidor DHCP (isc-dhcp-server)

Repartirá direcciones a las 4 VLANs de Santiago, incluyendo la de Management.

```bash
sudo apt install -y isc-dhcp-server
sudo nano /etc/default/isc-dhcp-server

```

*Modifica esta línea para indicar tu interfaz:*
`INTERFACESv4="ens3"`

```bash
sudo nano /etc/dhcp/dhcpd.conf

```

```text
authoritative;
default-lease-time 604800;
max-lease-time 604800;

# VLAN 210 - VENTAS SANTIAGO (/27)
subnet 10.1.0.0 netmask 255.255.255.224 {
  range 10.1.0.2 10.1.0.30;
  option routers 10.1.0.1;
  option domain-name-servers 10.1.0.34;
  option domain-name "netsecure.com.do";
}

# VLAN 220 - DATACENTER SANTIAGO (/27)
subnet 10.1.0.32 netmask 255.255.255.224 {
  range 10.1.0.35 10.1.0.62;
  option routers 10.1.0.33;
  option domain-name-servers 10.1.0.34;
  option domain-name "netsecure.com.do";
}

# VLAN 230 - ADMIN SANTIAGO (/28)
subnet 10.1.0.64 netmask 255.255.255.240 {
  range 10.1.0.66 10.1.0.78;
  option routers 10.1.0.65;
  option domain-name-servers 10.1.0.34;
  option domain-name "netsecure.com.do";
}

# VLAN 299 - MANAGEMENT SANTIAGO (/28)
subnet 10.1.1.0 netmask 255.255.255.240 {
  range 10.1.1.4 10.1.1.14;
  option routers 10.1.1.1;
  option domain-name-servers 10.1.0.34;
  option domain-name "netsecure.com.do";
}

```

```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server

```

---

### 4. Servidor DNS (Bind9)

Encargado de la resolución interna de `netsecure.com.do`.

```bash
sudo apt install -y bind9 bind9utils bind9-doc
sudo nano /etc/bind/named.conf.local

```

```text
zone "netsecure.com.do" {
    type master;
    file "/etc/bind/db.netsecure.com.do";
};

```

```bash
sudo cp /etc/bind/db.local /etc/bind/db.netsecure.com.do
sudo nano /etc/bind/db.netsecure.com.do

```

```text
$TTL    604800
@   IN  SOA ns1.netsecure.com.do. admin.netsecure.com.do. (
              4   ; Serial
         604800   ; Refresh
          86400   ; Retry
        2419200   ; Expire
         604800 ) ; Negative Cache TTL
;
@       IN  NS      ns1.netsecure.com.do.
ns1     IN  A       10.1.0.34
www     IN  A       1.0.0.26
mail    IN  A       10.1.0.34
@       IN  MX  10  mail.netsecure.com.do.
srv     IN  A       10.1.0.34
@       IN  A       1.0.0.26

```

```bash
sudo named-checkzone netsecure.com.do /etc/bind/db.netsecure.com.do
sudo systemctl restart bind9
sudo systemctl enable --now named

```

---

### 5. Servidor de Archivos FTP (vsftpd)

```bash
sudo apt install -y vsftpd
sudo nano /etc/vsftpd.conf

```

*Asegúrate de agregar/descomentar estas líneas al final del archivo:*

```text
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
chroot_local_user=YES
allow_writeable_chroot=YES
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100

```

```bash
sudo systemctl restart vsftpd
sudo systemctl enable vsftpd

```

---

### 6. Servidor de Correo (Postfix + Dovecot)

```bash
sudo apt install -y postfix dovecot-imapd dovecot-pop3d mailutils

```

*(Durante la instalación, selecciona **"Sitio de Internet"** y escribe `netsecure.com.do`)*

```bash
sudo postconf -e "myhostname = mail.netsecure.com.do"
sudo postconf -e "mydomain = netsecure.com.do"
sudo postconf -e "myorigin = /etc/mailname"
sudo postconf -e "inet_interfaces = all"
sudo postconf -e "mydestination = \$myhostname, netsecure.com.do, localhost"
echo "netsecure.com.do" | sudo tee /etc/mailname

# Configuración de Dovecot para usar buzones tipo Maildir
sudo sed -i 's/#mail_location = .*/mail_location = maildir:~\/Maildir/' /etc/dovecot/conf.d/10-mail.conf

sudo systemctl restart postfix dovecot
sudo systemctl enable postfix dovecot

```

---

### 7. Servidor RADIUS (FreeRADIUS)

Centraliza la autenticación (AAA) de todos los Routers y Switches Cisco de la topología.

```bash
sudo apt install -y freeradius freeradius-utils
sudo nano /etc/freeradius/3.0/clients.conf

```

*Agrega al final para permitir consultas desde toda tu red interna:*

```text
client netsecure-red {
    ipaddr     = 10.0.0.0/9
    secret     = Netsecure-EMP1
}

```

```bash
sudo nano /etc/freeradius/3.0/users

```

*Agrega al principio del archivo (antes de cualquier otra configuración):*

```text
isamar    Cleartext-Password := "Isamar2026*"
keury     Cleartext-Password := "Keury2026*"
sahir     Cleartext-Password := "Sahir2026*"
lizbeth   Cleartext-Password := "Lizbeth2026*"
starling  Cleartext-Password := "Starling2026*"
josue     Cleartext-Password := "Josue2026*"

```

```bash
sudo systemctl restart freeradius
sudo systemctl enable freeradius

```
