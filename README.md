# 🏢 Proyecto Final - NetSecure Solutions (CECOMPE)

![Topología de Red](Topologia.jpg)

## 📖 Introducción
NetSecure Solutions se dedicará a la venta o prestación de servicios en general, haciendo énfasis en el uso de Internet, Centros de atención al cliente (call centers) y las ventas directas. Esta organización de capital dominicano tendrá presencia a nivel nacional con sucursales en Santiago, La Romana, Puerto Plata, Barahona y su sede principal en Santo Domingo.

**CECOMPE (Centro de Cómputos Pelegrino)** es una compañía que se dedica a la elaboración de propuestas para soluciones de TI, la cual fue apoderada para que realizara el levantamiento correspondiente y presentara una propuesta tecnológica de comunicaciones unificadas acorde a la finalidad de la empresa y la disponibilidad de recursos.

Su grupo de la asignatura de Conmutación y Enrutamiento del ITLA ha sido elegido para implementar la propuesta y el diseño de CECOMPE en el emulador PNETLAB; emitir un informe de los resultados obtenidos.

El proyecto presenta un escenario completo para la implementación, sin embargo la misma puede ser modificado con justificación documentada. La empresa está dispuesta a implementar lo que sea necesario para el buen funcionamiento de la misma.

> **Nota de Despliegue de Servidores:** El **Servidor Web** está conectado y alojado directamente en la red del **ISP**, mientras que los demás servicios (DNS, DHCP, RADIUS, Correo) se encuentran alojados en el **Datacenter de la sede Santiago**.

---

## ⚙️ Aspectos Generales
- **Direccionamiento Privado** para toda la topología **`10.0.0.0/9`**, tomar en cuenta el crecimiento de la red 40% en 5 años.
- **Direccionamiento Publico `1.0.0.0/24`**
- Enrutamiento OSPF **Multiárea**
- Acceso remoto seguro para **todos** los dispositivos
- Implementa Seguridad en **todos** los dispositivos (Control de acceso, Seguridad de Puertos, Ataques de VLAN, Ataques DHCP, Ataques de ARP, Ataques de STP)
- Implementar DHCP por Sede **(En la Sede de Santiago usar un servidor del datacenter para que sea el server DHCP de esa Sede)**
- Implementar VPN dinámica con IPSec (DMVPN) para la comunicación entre las sucursales y la sede Principal
- Implementar ACL según el criterio del equipo

---

## 📍 Información Específica de las Sedes

### Sede Principal Central Santo Domingo
Controla el Gran Santo Domingo y la región Sur Central. Concentra la mayor demanda corporativa y gubernamental.
Crear Mínimo 4 Departamentos:
- **CALLCENTER-SD** (110 host + 40% a 5 años)
- **VENTAS-SD** (80 host + 40% a 5 años)
- **NOC-SOC-SD** (64 host + 40% a 5 años)
- **ADMIN-GERENCIA-SD** (51 host + 40% a 5 años)

*Implementar redundancia de capa 2 (EtherChannel y SpanningTree) y capa 3 (HSRP) donde aplique. En Santo Domingo la doble conexion con los SMW y el R son subredes de 2 host y ponerle una subred al EtherChannel.*

### Sede Santiago
Domina el Cibao Central. Es la segunda capital económica del país.
Crear Mínimo 3 Departamentos:
- **VENTAS-ST** (15 host + 40% a 5años)
- **DATACENTER-ST** (10 host + 40% a 5años)
- **ADMIN-ST** (5 host + 40% a 5años)

*Implementar en un Servidor cada uno de los siguientes servicios:*
- Servicio DNS (NombreEmpresa.com.do)
- Servicio DHCP (Debe asiganar ip a las Vlans de esa Sede)
- Servicio Web (URL:NombreEmpresa.com.do, se debe montar la pagina web de la empresa)
- Servicio Radius (Implementar un Servidor Radius, y crear un usuario para cada integarnte del grupo)
- Servicio De Correo (Dominio: NombreEmpresa.com.do, cree una cuenta para cada integrante)

### Sede Romana
Domina la Región Este. Permite atender el polo turístico e industrial más grande del país.
Crear Mínimo 3 Departamentos:
- **VENTAS-ATENCION-ROM** (52 host + 40% a 5 años)
- **LOGISTICA-ROM** (25 host + 40% a 5 años)
- **RRHH-ROM** (7 host + 40% a 5 años)
*(DHCP en su respectivo Router)*

### Sede Barahona
Controla el Suroeste. Posición estratégica ante el desarrollo de Pedernales-Cabo Rojo.
Crear Mínimo 3 Departamentos:
- **VENTAS-NEGOCIOS-BAR** (52 host + 40% a 5 años)
- **SOPORTE-TECNICO-BAR** (25 host + 40% a 5 años)
- **ADMINISTRACION-BAR** (7 host + 40% a 5 años)
*(DHCP en su respectivo Router)*

### Sede Puerto Plata
Asegura el Cibao Norte y la Costa Atlántica.
Crear Mínimo 3 Departamentos:
- **VENTAS-COMERCIAL-PPL** (52 host + 40% a 5 años)
- **MARKETING-PPL** (25 host + 40% a 5 años)
- **CONTABILIDAD-PPL** (7 host + 40% a 5 años)
*(DHCP en su respectivo Router)*

### Sede San Francisco de Macorís
Cubre el Nordeste. Centro de operaciones productoras con alta demanda de digitalización.
Crear Mínimo 3 Departamentos:
- **VENTAS-REGIONAL-SFM** (52 host + 40% a 5 años)
- **OPERACIONES-SFM** (25 host + 40% a 5 años)
- **LEGAL-CUMPLIMIENTO-SFM** (7 host + 40% a 5 años)
*(DHCP en su respectivo Router)*

---

## 📊 Tablas de Direccionamiento

### Direccionamiento Público
| Descripción | Red | Mascara | Primera utilizable | Broadcast |
| :--- | :--- | :--- | :--- | :--- |
| Santo Domingo ↔ ISP | `1.0.0.0/30` | `255.255.255.252` | `1.0.0.1` | `1.0.0.3` |
| Santiago ↔ ISP | `1.0.0.4/30` | `255.255.255.252` | `1.0.0.5` | `1.0.0.7` |
| La Romana ↔ ISP | `1.0.0.8/30` | `255.255.255.252` | `1.0.0.9` | `1.0.0.11` |
| Puerto Plata ↔ ISP | `1.0.0.12/30` | `255.255.255.252` | `1.0.0.13` | `1.0.0.15` |
| Barahona ↔ ISP | `1.0.0.16/30` | `255.255.255.252` | `1.0.0.17` | `1.0.0.19` |
| San Francisco de Macorís ↔ ISP | `1.0.0.20/30` | `255.255.255.252` | `1.0.0.21` | `1.0.0.23` |
| NAT estático — Web/Correo | `1.0.0.24/29` | `255.255.255.248` | `1.0.0.25` | `1.0.0.31` |

### DMVPN/IPSec - 10.7.0.0/28
| Dispositivo | Rol | IP del túnel |
| :--- | :--- | :--- |
| R-CORE — Santo Domingo | Hub | `10.7.0.1` |
| Router Santiago | Spoke | `10.7.0.2` |
| Router La Romana | Spoke | `10.7.0.3` |
| Router Puerto Plata | Spoke | `10.7.0.4` |
| Router Barahona | Spoke | `10.7.0.5` |
| Router San Francisco de Macorís | Spoke | `10.7.0.6` |

* **Red:** `10.7.0.0/28`
* **Máscara:** `255.255.255.240`
* **Primera IP utilizable:** `10.7.0.1` | **Última:** `10.7.0.14` | **Broadcast:** `10.7.0.15`

### Direccionamiento Privado por Sede

<details>
<summary><b>Santo Domingo</b></summary>

| VLAN | Nombre | Propósito | Hosts +40% | Subred | Primera IP | Última IP | Broadcast |
| ---: | :--- | :--- | ---: | :--- | :--- | :--- | :--- |
| 100 | DEFAULT-SD | VLAN nativa | — | — | — | — | — |
| 110 | CALLCENTER-SD | Departamento 1 | 154 | `10.0.0.0/24` | `10.0.0.1` | `10.0.0.254` | `10.0.0.255` |
| 120 | VENTAS-SD | Departamento 2 | 112 | `10.0.1.0/25` | `10.0.1.1` | `10.0.1.126` | `10.0.1.127` |
| 130 | NOC-SOC-SD | Departamento 3 | 90 | `10.0.1.128/25` | `10.0.1.129` | `10.0.1.254` | `10.0.1.255` |
| 140 | ADMIN-GERENCIA-SD | Departamento 4 | 72 | `10.0.2.0/25` | `10.0.2.1` | `10.0.2.126` | `10.0.2.127` |
| 199 | MGMT-SD | Gestión/OOB | — | `10.0.3.0/28` | `10.0.3.1` | `10.0.3.14` | `10.0.3.15` |

**Enlaces de Infraestructura Interna:**
* R-CORE ↔ Switch núcleo 1: `10.6.0.0/30` (`.1` al `.2`)
* R-CORE ↔ Switch núcleo 2: `10.6.0.4/30` (`.5` al `.6`)
* EtherChannel L3 R8 ↔ R9: `10.6.0.8/30` (`.9` al `.10`)
</details>

<details>
<summary><b>Santiago</b></summary>

| VLAN | Nombre | Propósito | Hosts +40% | Subred | Primera IP | Última IP | Broadcast |
| ---: | :--- | :--- | ---: | :--- | :--- | :--- | :--- |
| 200 | DEFAULT-ST | VLAN nativa | — | — | — | — | — |
| 210 | VENTAS-ST | Ventas | 21 | `10.1.0.0/27` | `10.1.0.1` | `10.1.0.30` | `10.1.0.31` |
| 220 | DATACENTER-ST | DNS/DHCP/Web/RADIUS | 14 | `10.1.0.32/27` | `10.1.0.33` | `10.1.0.62` | `10.1.0.63` |
| 230 | ADMIN-ST | Administración | 7 | `10.1.0.64/28` | `10.1.0.65` | `10.1.0.78` | `10.1.0.79` |
| 299 | MGMT-ST | Gestión/OOB | — | `10.1.1.0/28` | `10.1.1.1` | `10.1.1.14` | `10.1.1.15` |

**Servidor Data Center:** `10.1.0.34` (Gateway: `10.1.0.33`)
</details>

<details>
<summary><b>La Romana</b></summary>

| VLAN | Nombre | Propósito | Hosts +40% | Subred | Primera IP | Última IP | Broadcast |
| ---: | :--- | :--- | ---: | :--- | :--- | :--- | :--- |
| 300 | DEFAULT-ROM | VLAN nativa | — | — | — | — | — |
| 310 | VENTAS-ATENCION-ROM | Departamento 1 | 73 | `10.2.0.0/25` | `10.2.0.1` | `10.2.0.126` | `10.2.0.127` |
| 320 | LOGISTICA-ROM | Departamento 2 | 35 | `10.2.0.128/26` | `10.2.0.129` | `10.2.0.190` | `10.2.0.191` |
| 330 | RRHH-ROM | Departamento 3 | 10 | `10.2.0.192/28` | `10.2.0.193` | `10.2.0.206` | `10.2.0.207` |
| 399 | MGMT-ROM | Gestión/OOB | — | `10.2.1.0/28` | `10.2.1.1` | `10.2.1.14` | `10.2.1.15` |
</details>

<details>
<summary><b>Puerto Plata</b></summary>

| VLAN | Nombre | Propósito | Hosts +40% | Subred | Primera IP | Última IP | Broadcast |
| ---: | :--- | :--- | ---: | :--- | :--- | :--- | :--- |
| 400 | DEFAULT-PPL | VLAN nativa | — | — | — | — | — |
| 410 | VENTAS-COMERCIAL-PPL | Departamento 1 | 73 | `10.3.0.0/25` | `10.3.0.1` | `10.3.0.126` | `10.3.0.127` |
| 420 | MARKETING-PPL | Departamento 2 | 35 | `10.3.0.128/26` | `10.3.0.129` | `10.3.0.190` | `10.3.0.191` |
| 430 | CONTABILIDAD-PPL | Departamento 3 | 10 | `10.3.0.192/28` | `10.3.0.193` | `10.3.0.206` | `10.3.0.207` |
| 499 | MGMT-PPL | Gestión/OOB | — | `10.3.1.0/28` | `10.3.1.1` | `10.3.1.14` | `10.3.1.15` |
</details>

<details>
<summary><b>Barahona</b></summary>

| VLAN | Nombre | Propósito | Hosts +40% | Subred | Primera IP | Última IP | Broadcast |
| ---: | :--- | :--- | ---: | :--- | :--- | :--- | :--- |
| 500 | DEFAULT-BAR | VLAN nativa | — | — | — | — | — |
| 510 | VENTAS-NEGOCIOS-BAR | Departamento 1 | 73 | `10.4.0.0/25` | `10.4.0.1` | `10.4.0.126` | `10.4.0.127` |
| 520 | SOPORTE-TECNICO-BAR | Departamento 2 | 35 | `10.4.0.128/26` | `10.4.0.129` | `10.4.0.190` | `10.4.0.191` |
| 530 | ADMINISTRACION-BAR | Departamento 3 | 10 | `10.4.0.192/28` | `10.4.0.193` | `10.4.0.206` | `10.4.0.207` |
| 599 | MGMT-BAR | Gestión/OOB | — | `10.4.1.0/28` | `10.4.1.1` | `10.4.1.14` | `10.4.1.15` |
</details>

<details>
<summary><b>San Francisco de Macorís</b></summary>

| VLAN | Nombre | Propósito | Hosts +40% | Subred | Primera IP | Última IP | Broadcast |
| ---: | :--- | :--- | ---: | :--- | :--- | :--- | :--- |
| 600 | DEFAULT-SFM | VLAN nativa | — | — | — | — | — |
| 610 | VENTAS-REGIONAL-SFM | Departamento 1 | 73 | `10.5.0.0/25` | `10.5.0.1` | `10.5.0.126` | `10.5.0.127` |
| 620 | OPERACIONES-SFM | Departamento 2 | 35 | `10.5.0.128/26` | `10.5.0.129` | `10.5.0.190` | `10.5.0.191` |
| 630 | LEGAL-CUMPLIMIENTO-SFM | Departamento 3 | 10 | `10.5.0.192/28` | `10.5.0.193` | `10.5.0.206` | `10.5.0.207` |
| 699 | MGMT-SFM | Gestión/OOB | — | `10.5.1.0/28` | `10.5.1.1` | `10.5.1.14` | `10.5.1.15` |
</details>

---

## 💻 Configuraciones (Scripts)

<details>
<summary><b>Router ISP-NUBE</b></summary>

```text
enable
configure terminal
!
hostname ISP-NUBE
no ip domain lookup
service timestamps debug datetime msec
service timestamps log datetime msec
ip cef
no ipv6 cef

! ENLACE SANTO DOMINGO
interface Ethernet0/0
 description ENLACE-A-SANTO-DOMINGO-HUB
 ip address 1.0.0.1 255.255.255.252
 ip nat inside
 no shutdown
exit

! ENLACE PUERTO PLATA
interface Ethernet0/1
 description ENLACE-A-PUERTO-PLATA
 ip address 1.0.0.13 255.255.255.252
 ip nat inside
 no shutdown
exit

! ENLACE SANTIAGO
interface Ethernet0/2
 description ENLACE-A-SANTIAGO
 ip address 1.0.0.5 255.255.255.252
 ip nat inside
 duplex full
 no shutdown
exit

! ENLACE LA ROMANA
interface Ethernet0/3
 description ENLACE-A-LA-ROMANA
 ip address 1.0.0.9 255.255.255.252
 ip nat inside
 no shutdown
exit

! ENLACE SAN FRANCISCO DE MACORIS
interface Ethernet1/0
 description ENLACE-A-SAN-FRANCISCO-MACORIS
 ip address 1.0.0.21 255.255.255.252
 ip nat inside
 no shutdown
exit

! ENLACE BARAHONA
interface Ethernet1/1
 description ENLACE-A-BARAHONA
 ip address 1.0.0.17 255.255.255.252
 ip nat inside
 no shutdown
exit

! ENLACE WEB SERVER
interface Ethernet2/0
 description ENLACE-A-WEB-SERVER
 ip address 1.0.0.25 255.255.255.248
 ip nat inside
 no shutdown
exit

! ENLACE ALTICE CLOUD / INTERNET
interface Ethernet3/0
 description ENLACE-A-ALTICE-CLOUD
 ip address dhcp
 ip nat outside
 no shutdown
exit

! NAT
access-list 100 permit ip any any
ip nat inside source list 100 interface Ethernet3/0 overload

! RUTAS
ip route 0.0.0.0 0.0.0.0 10.0.137.1
ip route 10.1.0.0 255.255.254.0 1.0.0.6
ip route 10.1.0.32 255.255.255.224 1.0.0.6

! SEGURIDAD
no ip http server
no ip http secure-server

end
write memory
```
</details>

<details>
<summary><b>Router R-BAR (Barahona)</b></summary>

```text
enable
configure terminal
!
hostname R-BAR
no ip domain lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1

ip cef
no ipv6 cef

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (BARAHONA)
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

! INTERFAZ WAN
interface Ethernet0/0
 description ENLACE-WAN-ISP
 ip address 1.0.0.18 255.255.255.252
 ip nat outside
 ip virtual-reassembly in
 no shutdown
exit

! TRUNK HACIA SWD-BAR-1
interface Ethernet0/1
 description TRUNK-A-SWD-BAR-1
 no ip address
 duplex full
 no shutdown
exit

interface Ethernet0/1.510
 description GATEWAY-VENTAS
 encapsulation dot1Q 510
 ip address 10.4.0.1 255.255.255.128
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 5
exit

interface Ethernet0/1.520
 description GATEWAY-SOPORTE
 encapsulation dot1Q 520
 ip address 10.4.0.129 255.255.255.192
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 5
exit

interface Ethernet0/1.530
 description GATEWAY-ADMIN
 encapsulation dot1Q 530
 ip address 10.4.0.193 255.255.255.240
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 5
exit

interface Ethernet0/1.599
 description GATEWAY-MGMT
 encapsulation dot1Q 599
 ip address 10.4.1.1 255.255.255.240
 ip nat inside
 ip virtual-reassembly in
 ip tcp adjust-mss 1360
 ip ospf 21 area 5
exit

! DHCP
ip dhcp excluded-address 10.4.0.1 10.4.0.3
ip dhcp excluded-address 10.4.0.129 10.4.0.131
ip dhcp excluded-address 10.4.0.193 10.4.0.195
ip dhcp excluded-address 10.4.1.1 10.4.1.3

ip dhcp pool VLAN510-VENTAS
 network 10.4.0.0 255.255.255.128
 default-router 10.4.0.1
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN520-SOPORTE
 network 10.4.0.128 255.255.255.192
 default-router 10.4.0.129
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN530-ADMIN
 network 10.4.0.192 255.255.255.240
 default-router 10.4.0.193
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

ip dhcp pool VLAN599-MGMT
 network 10.4.1.0 255.255.255.240
 default-router 10.4.1.1
 dns-server 10.1.0.34
 domain-name netsecure.com.do
 lease 7
exit

! OSPF
router ospf 21
 router-id 5.5.5.5
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface Tunnel1
exit

ip route 0.0.0.0 0.0.0.0 1.0.0.17

! NAT
ip access-list extended NAT-ACL-BAR
 deny ip 10.4.0.0 0.0.255.255 10.0.0.0 0.255.255.255
 permit ip 10.4.0.0 0.0.255.255 any
exit
ip nat inside source list NAT-ACL-BAR interface Ethernet0/0 overload

! VPN DMVPN/IPSEC
crypto isakmp policy 21
 encr aes
 hash sha512
 authentication pre-share
 group 14
exit
crypto isakmp key Cisco123 address 1.0.0.2

crypto ipsec transform-set TS-VPN esp-aes esp-sha512-hmac
 mode transport
exit
crypto ipsec profile PF-VPN
 set transform-set TS-VPN
exit

interface Tunnel1
 description DMVPN-SPOKE-BARAHONA
 ip address 10.7.0.5 255.255.255.240
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

no ip http server
no ip http secure-server
end
write memory
```
</details>

<details>
<summary><b>Switch Distribución SWD-BAR-1 (Barahona)</b></summary>

```text
enable
configure terminal
!
hostname SWD-BAR-1
no ip domain-lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain-name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1
no ip routing

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (BARAHONA)
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

! VLANS
vlan 500
 name DEFAULT-BAR
vlan 510
 name VENTAS-NEGOCIOS-BAR
vlan 520
 name SOPORTE-TECNICO-BAR
vlan 530
 name ADMINISTRACION-BAR
vlan 599
 name MGMT-BAR
exit

! SPANNING TREE
spanning-tree mode pvst
spanning-tree vlan 500,510,520,530,599 priority 24576

! TRUNK HACIA R-BAR
interface Ethernet0/0
 description TRUNK-A-R-BAR
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 500,510,520,530,599
 switchport trunk native vlan 500
 switchport nonegotiate
 duplex full
 no shutdown
exit

! ETHERCHANNEL HACIA SWA-BAR-1
interface range Ethernet0/1-3
 description ETHERCHANNEL-L2-SWA-BAR-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 500,510,520,530,599
 switchport trunk native vlan 500
 switchport nonegotiate
 duplex full
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 description TRUNK-A-SWA-BAR-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 500,510,520,530,599
 switchport trunk native vlan 500
 switchport nonegotiate
 no shutdown
exit

! MANAGEMENT
interface Vlan599
 description MANAGEMENT-SWD-BAR-1
 ip address 10.4.1.2 255.255.255.240
 no shutdown
exit

ip default-gateway 10.4.1.1
ip radius source-interface Vlan599
no ip http server
no ip http secure-server
end
write memory
```
</details>

<details>
<summary><b>Switch Acceso SWA-BAR-1 (Barahona)</b></summary>

```text
enable
configure terminal
!
hostname SWA-BAR-1
no ip domain-lookup
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption

ip domain-name netsecure.com.do
enable secret Netsecure-EMP1
username admin privilege 15 secret Netsecure-EMP1

banner motd #
*********************************************************************
NETSECURE SOLUTIONS (BARAHONA)
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

vlan 500
 name DEFAULT-BAR
vlan 510
 name VENTAS-NEGOCIOS-BAR
vlan 520
 name SOPORTE-TECNICO-BAR
vlan 530
 name ADMINISTRACION-BAR
vlan 599
 name MGMT-BAR
exit

spanning-tree mode pvst

! ETHERCHANNEL
interface range Ethernet0/1-3
 description ETHERCHANNEL-L2-SWD-BAR-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 500,510,520,530,599
 switchport trunk native vlan 500
 switchport nonegotiate
 channel-group 1 mode active
 no shutdown
exit

interface Port-channel1
 description TRUNK-A-SWD-BAR-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 500,510,520,530,599
 switchport trunk native vlan 500
 switchport nonegotiate
 no shutdown
exit

! PUERTO MANAGEMENT
interface Ethernet0/0
 description MGMT-BAR
 switchport mode access
 switchport access vlan 599
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

! VENTAS / NEGOCIOS
interface range Ethernet1/0-3
 description VENTAS-NEGOCIOS-BAR
 switchport mode access
 switchport access vlan 510
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

! SOPORTE TECNICO
interface range Ethernet2/0-3
 description SOPORTE-TECNICO-BAR
 switchport mode access
 switchport access vlan 520
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

! ADMINISTRACION
interface range Ethernet3/0-3
 description ADMINISTRACION-BAR
 switchport mode access
 switchport access vlan 530
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
exit

! MANAGEMENT SVI
interface Vlan599
 description MANAGEMENT-SWA-BAR-1
 ip address 10.4.1.3 255.255.255.240
 no shutdown
exit
ip default-gateway 10.4.1.1
ip radius source-interface Vlan599
no ip http server
no ip http secure-server

! DHCP SNOOPING
ip dhcp snooping vlan 510,520,530
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
interface range Ethernet2/0-3
 ip dhcp snooping limit rate 15
exit
interface range Ethernet3/0-3
 ip dhcp snooping limit rate 15
exit

! DAI / ARP INSPECTION
ip arp inspection vlan 510,520,530

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

