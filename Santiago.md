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

